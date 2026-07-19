# Bunny Game All Scripts

- Proyecto: Bunny Game
- Carpeta origen: `Assets/Project`
- Total scripts: 16
- Exportado: 2026-07-19 13:36:27

## Índice

- `Assets/Project/Scripts/Combat/CombatHUD.cs`
- `Assets/Project/Scripts/Combat/CombatManager.cs`
- `Assets/Project/Scripts/Combat/CombatTile.cs`
- `Assets/Project/Scripts/Combat/CombatZone.cs`
- `Assets/Project/Scripts/Combat/CombatZoneMember.cs`
- `Assets/Project/Scripts/Combat/TurnSystem.cs`
- `Assets/Project/Scripts/Grid/CellVisual.cs`
- `Assets/Project/Scripts/Grid/CoverDirection.cs`
- `Assets/Project/Scripts/Grid/CoverType.cs`
- `Assets/Project/Scripts/Grid/GridCell.cs`
- `Assets/Project/Scripts/Grid/GridManager.cs`
- `Assets/Project/Scripts/Player/IPlayerState.cs`
- `Assets/Project/Scripts/Player/PlayerCombatModeState.cs`
- `Assets/Project/Scripts/Player/PlayerExplorationState.cs`
- `Assets/Project/Scripts/Player/PlayerStateMachine.cs`
- `Assets/Project/Scripts/Units/UnitController.cs`

=== CombatHUD.cs ===
```csharp
using UnityEngine;
using TMPro;
using UnityEngine.UI;

public class CombatHUD : MonoBehaviour
{
    [Header("Panels")]
    [SerializeField] private Image movePanel;
    [SerializeField] private Image shootPanel;
    [SerializeField] private Image cancelPanel;
    [SerializeField] private Image endTurnPanel;

    [Header("Labels")]
    [SerializeField] private TMP_Text moveLabel;
    [SerializeField] private TMP_Text shootLabel;
    [SerializeField] private TMP_Text cancelLabel;
    [SerializeField] private TMP_Text endTurnLabel;
    [SerializeField] private TMP_Text statusText;

    [Header("Colors")]
    [SerializeField] private Color inactiveColor = new Color(0.15f, 0.15f, 0.15f, 0.75f);
    [SerializeField] private Color activeColor = new Color(0.2f, 0.7f, 1f, 0.95f);
    [SerializeField] private Color disabledColor = new Color(0.1f, 0.1f, 0.1f, 0.35f);

    public enum HUDMode
    {
        None,
        Move,
        Shoot
    }

    public void SetVisible(bool value)
    {
        gameObject.SetActive(value);
    }

    public void Refresh(bool hasSelectedUnit, bool canAct, HUDMode mode)
    {
        ResetPanels();

        if (!hasSelectedUnit)
        {
            statusText.text = "Selecciona una unidad";
            SetPanelState(movePanel, false);
            SetPanelState(shootPanel, false);
            SetPanelState(cancelPanel, false);
            SetPanelState(endTurnPanel, true);
            return;
        }

        if (!canAct)
        {
            statusText.text = "Unidad sin acciones";
            SetPanelState(movePanel, false);
            SetPanelState(shootPanel, false);
            SetPanelState(cancelPanel, false);
            SetPanelState(endTurnPanel, true);
            return;
        }

        statusText.text = "Elige una acci�n";

        SetPanelState(movePanel, true);
        SetPanelState(shootPanel, true);
        SetPanelState(cancelPanel, true);
        SetPanelState(endTurnPanel, true);

        switch (mode)
        {
            case HUDMode.Move:
                statusText.text = "Modo mover activo";
                HighlightPanel(movePanel);
                break;

            case HUDMode.Shoot:
                statusText.text = "Modo disparo activo";
                HighlightPanel(shootPanel);
                break;

            case HUDMode.None:
                statusText.text = "Esperando acci�n";
                break;
        }
    }

    private void ResetPanels()
    {
        movePanel.color = inactiveColor;
        shootPanel.color = inactiveColor;
        cancelPanel.color = inactiveColor;
        endTurnPanel.color = inactiveColor;
    }

    private void HighlightPanel(Image panel)
    {
        panel.color = activeColor;
    }

    private void SetPanelState(Image panel, bool enabled)
    {
        panel.color = enabled ? inactiveColor : disabledColor;
    }
}
```

=== CombatManager.cs ===
```csharp
using UnityEngine;
using System.Collections;
using System.Collections.Generic;
using UnityEngine.InputSystem;

public class CombatManager : MonoBehaviour
{
    private enum ActionMode
    {
        None,
        Move,
        Shoot
    }

    [SerializeField] private GridManager gridManager;
    [SerializeField] private TurnSystem turnSystem;
    [SerializeField] private UnitController playerUnitPrefab;
    [SerializeField] private UnitController enemyUnitPrefab;
    [SerializeField] private Transform unitsParent;
    [SerializeField] private CombatHUD combatHUD;

    private UnitController playerUnit;
    private UnitController enemyUnit;
    private UnitController selectedUnit;

    private List<GridCell> currentReachableCells = new List<GridCell>();
    private List<GridCell> currentShootableCells = new List<GridCell>();

    private bool isBusy;
    private ActionMode currentMode = ActionMode.None;

    private CombatZone activeCombatZone;

    public bool CombatFinished { get; private set; }

    public void SetCombatZone(CombatZone combatZone)
    {
        activeCombatZone = combatZone;
    }

    public void BeginCombat()
    {
        Debug.Log("Begin Combat");

        CombatFinished = false;
        isBusy = false;
        currentMode = ActionMode.None;

        playerUnit = null;
        enemyUnit = null;
        selectedUnit = null;
        currentReachableCells.Clear();
        currentShootableCells.Clear();

        if (activeCombatZone == null)
        {
            Debug.LogError("[CombatManager] No active combat zone assigned.");
            return;
        }

        if (activeCombatZone.GridManager == null)
        {
            Debug.LogError("[CombatManager] La CombatZone no tiene GridManager asignado.");
            return;
        }

        if (activeCombatZone.TilesRoot == null)
        {
            Debug.LogError("[CombatManager] La CombatZone no tiene TilesRoot asignado.");
            return;
        }

        if (activeCombatZone.PlayerUnitsRoot == null)
        {
            Debug.LogError("[CombatManager] La CombatZone no tiene PlayerUnitsRoot asignado.");
            return;
        }

        if (activeCombatZone.EnemyUnitsRoot == null)
        {
            Debug.LogError("[CombatManager] La CombatZone no tiene EnemyUnitsRoot asignado.");
            return;
        }

        gridManager = activeCombatZone.GridManager;
        gridManager.BuildGridFromScene(activeCombatZone.TilesRoot);

        if (turnSystem == null)
        {
            Debug.LogError("[CombatManager] TurnSystem no est� asignado.");
            return;
        }

        turnSystem.StartTurns();

        UnitController[] playerUnits = activeCombatZone.PlayerUnitsRoot.GetComponentsInChildren<UnitController>(true);
        UnitController[] enemyUnits = activeCombatZone.EnemyUnitsRoot.GetComponentsInChildren<UnitController>(true);

        if (playerUnits.Length > 0)
        {
            playerUnit = playerUnits[0];

            CombatZoneMember playerMember = playerUnit.GetComponent<CombatZoneMember>();
            if (playerMember == null)
            {
                Debug.LogError($"[CombatManager] {playerUnit.name} no tiene CombatZoneMember.");
            }
            else
            {
                Debug.Log($"[CombatManager] PlayerMember.StartingCoordinates = {playerMember.StartingCoordinates}");

                GridCell playerSpawn = gridManager.GetCell(playerMember.StartingCoordinates);
                if (playerSpawn != null)
                {
                    Debug.Log($"[CombatManager] Player spawn encontrado: {playerSpawn.Coordinates} en world {playerSpawn.transform.position}");
                    playerUnit.Place(playerSpawn, gridManager);
                    playerUnit.StartTurn();
                }
                else
                {
                    Debug.LogError($"[CombatManager] No existe GridCell para el jugador en {playerMember.StartingCoordinates}");
                }
            }
        }
        else
        {
            Debug.LogWarning("[CombatManager] No se encontraron unidades del jugador en PlayerUnitsRoot.");
        }

        if (enemyUnits.Length > 0)
        {
            enemyUnit = enemyUnits[0];

            CombatZoneMember enemyMember = enemyUnit.GetComponent<CombatZoneMember>();
            if (enemyMember == null)
            {
                Debug.LogError($"[CombatManager] {enemyUnit.name} no tiene CombatZoneMember.");
            }
            else
            {
                Debug.Log($"[CombatManager] EnemyMember.StartingCoordinates = {enemyMember.StartingCoordinates}");

                GridCell enemySpawn = gridManager.GetCell(enemyMember.StartingCoordinates);
                if (enemySpawn != null)
                {
                    Debug.Log($"[CombatManager] Enemy spawn encontrado: {enemySpawn.Coordinates} en world {enemySpawn.transform.position}");
                    enemyUnit.Place(enemySpawn, gridManager);
                    enemyUnit.StartTurn();
                }
                else
                {
                    Debug.LogError($"[CombatManager] No existe GridCell para el enemigo en {enemyMember.StartingCoordinates}");
                }
            }
        }
        else
        {
            Debug.LogWarning("[CombatManager] No se encontraron unidades enemigas en EnemyUnitsRoot.");
        }

        if (combatHUD != null)
        {
            combatHUD.SetVisible(true);
            RefreshHUD();
        }
    }

    public void EndCombat()
    {
        if (combatHUD != null)
            combatHUD.SetVisible(false);

        Debug.Log("End Combat");
    }

    private void Update()
    {
        if (CombatFinished || isBusy)
            return;

        if (turnSystem == null || !turnSystem.IsPlayerTurn())
            return;

        if (Keyboard.current != null)
        {
            if (Keyboard.current.mKey.wasPressedThisFrame && selectedUnit != null && selectedUnit.HasActionPoints())
            {
                EnterMoveMode();
            }

            if (Keyboard.current.fKey.wasPressedThisFrame && selectedUnit != null && selectedUnit.HasActionPoints())
            {
                EnterShootMode();
            }

            if (Keyboard.current.escapeKey.wasPressedThisFrame)
            {
                currentMode = ActionMode.None;
                currentReachableCells.Clear();
                currentShootableCells.Clear();
                RefreshSelectionVisuals();
            }

            if (Keyboard.current.spaceKey.wasPressedThisFrame)
            {
                EndPlayerTurn();
            }
        }

        if (Mouse.current != null && Mouse.current.leftButton.wasPressedThisFrame)
        {
            HandleLeftClick();
        }
    }

    private void HandleLeftClick()
    {
        if (Camera.main == null)
        {
            Debug.LogError("[CombatManager] No existe Camera.main.");
            return;
        }

        Ray ray = Camera.main.ScreenPointToRay(Mouse.current.position.ReadValue());

        if (!Physics.Raycast(ray, out RaycastHit hit, 100f, Physics.DefaultRaycastLayers, QueryTriggerInteraction.Ignore))
        {
            Debug.Log("[CombatManager] El raycast no ha golpeado nada.");
            return;
        }

        Debug.Log($"[CombatManager] Click sobre: {hit.collider.name}");

        UnitController clickedUnit = hit.collider.GetComponentInParent<UnitController>();
        if (clickedUnit != null)
        {
            Debug.Log($"[CombatManager] He encontrado UnitController en: {clickedUnit.name}, IsEnemy={clickedUnit.IsEnemy}");

            if (!clickedUnit.IsEnemy)
            {
                SelectUnit(clickedUnit);
                return;
            }

            if (currentMode == ActionMode.Shoot)
            {
                TryShootEnemy(clickedUnit);
                return;
            }
        }

        GridCell clickedCell = hit.collider.GetComponentInParent<GridCell>();
        if (clickedCell != null)
        {
            Debug.Log($"[CombatManager] He encontrado GridCell en: {clickedCell.name}, coords={clickedCell.Coordinates}");

            if (currentMode == ActionMode.Move)
            {
                TryMoveSelectedUnit(clickedCell);
                return;
            }
        }

        Debug.Log("[CombatManager] El click no encontr� ni UnitController ni GridCell �tiles.");
    }

    private void SelectUnit(UnitController unit)
    {
        if (unit == null)
            return;

        Debug.Log($"[CombatManager] Intentando seleccionar unidad: {unit.name}");

        ClearSelection();

        selectedUnit = unit;
        selectedUnit.Select();
        currentMode = ActionMode.None;

        Debug.Log($"[CombatManager] Unidad seleccionada: {selectedUnit.name}, GridPosition={selectedUnit.GridPosition}, ActionPoints={selectedUnit.ActionPoints}");

        RefreshSelectionVisuals();
        RefreshHUD();
    }

    private void EnterMoveMode()
    {
        if (selectedUnit == null)
        {
            Debug.LogWarning("[CombatManager] No hay unidad seleccionada para mover.");
            return;
        }

        Debug.Log($"[CombatManager] EnterMoveMode con unidad {selectedUnit.name} en {selectedUnit.GridPosition}");

        currentMode = ActionMode.Move;
        currentReachableCells = gridManager.GetReachableCells(selectedUnit);
        currentShootableCells.Clear();

        Debug.Log($"[CombatManager] Modo mover activado. Celdas alcanzables: {currentReachableCells.Count}");

        foreach (GridCell cell in currentReachableCells)
        {
            Debug.Log($"[CombatManager] Alcanzable -> {cell.Coordinates}");
        }

        RefreshSelectionVisuals();
        RefreshHUD();
    }

    private void EnterShootMode()
    {
        if (selectedUnit == null)
        {
            Debug.LogWarning("[CombatManager] No hay unidad seleccionada para disparar.");
            return;
        }

        Debug.Log($"[CombatManager] EnterShootMode con unidad {selectedUnit.name} en {selectedUnit.GridPosition}");

        currentMode = ActionMode.Shoot;
        currentShootableCells = gridManager.GetShootableCells(selectedUnit);
        currentReachableCells.Clear();

        Debug.Log($"[CombatManager] Modo disparo activado. Celdas atacables: {currentShootableCells.Count}");

        foreach (GridCell cell in currentShootableCells)
        {
            Debug.Log($"[CombatManager] Atacable -> {cell.Coordinates}");
        }

        RefreshSelectionVisuals();
        RefreshHUD();
    }

    private void RefreshSelectionVisuals()
    {
        if (gridManager == null)
            return;

        gridManager.ResetAllCellVisuals();

        if (selectedUnit == null)
        {
            RefreshHUD();
            return;
        }

        GridCell unitCell = gridManager.GetCell(selectedUnit.GridPosition);

        if (unitCell == null)
        {
            Debug.LogError($"[CombatManager] No encuentro la GridCell de la unidad seleccionada {selectedUnit.name} en {selectedUnit.GridPosition}");
            RefreshHUD();
            return;
        }

        unitCell.SetSelectedVisual();

        if (currentMode == ActionMode.Move)
            gridManager.HighlightCells(currentReachableCells);

        if (currentMode == ActionMode.Shoot)
            gridManager.HighlightAttackableCells(currentShootableCells);

        RefreshHUD();
    }

    private void TryMoveSelectedUnit(GridCell targetCell)
    {
        if (selectedUnit == null)
        {
            Debug.LogWarning("[CombatManager] No hay unidad seleccionada.");
            return;
        }

        if (targetCell == null)
        {
            Debug.LogWarning("[CombatManager] targetCell es null.");
            return;
        }

        Debug.Log($"[CombatManager] Intentando mover a {targetCell.Coordinates}");
        Debug.Log($"[CombatManager] Unidad seleccionada en {selectedUnit.GridPosition}");
        Debug.Log($"[CombatManager] Celdas alcanzables: {currentReachableCells.Count}");

        foreach (GridCell cell in currentReachableCells)
        {
            Debug.Log($"[CombatManager] Alcanzable -> {cell.Coordinates}");
        }

        bool isReachable = ContainsCellWithCoordinates(currentReachableCells, targetCell.Coordinates);

        if (!isReachable)
        {
            Debug.LogWarning($"[CombatManager] La celda {targetCell.Coordinates} NO est� en currentReachableCells.");
            return;
        }

        GridCell currentCell = gridManager.GetCell(selectedUnit.GridPosition);

        if (currentCell == null)
        {
            Debug.LogError("[CombatManager] La celda actual de la unidad seleccionada es null.");
            return;
        }

        StartCoroutine(MoveSelectedUnitRoutine(currentCell, targetCell));
    }

    private IEnumerator MoveSelectedUnitRoutine(GridCell fromCell, GridCell toCell)
    {
        isBusy = true;

        yield return selectedUnit.MoveTo(fromCell, toCell, gridManager);

        selectedUnit.SpendActionPoint();
        currentMode = ActionMode.None;
        currentReachableCells.Clear();
        currentShootableCells.Clear();

        RefreshSelectionVisuals();

        isBusy = false;
    }

    private void TryShootEnemy(UnitController target)
    {
        if (selectedUnit == null)
        {
            Debug.LogWarning("[CombatManager] No hay unidad seleccionada para disparar.");
            return;
        }

        if (target == null)
        {
            Debug.LogWarning("[CombatManager] El objetivo es null.");
            return;
        }

        GridCell targetCell = gridManager.GetCell(target.GridPosition);

        if (targetCell == null)
        {
            Debug.LogError($"[CombatManager] No encuentro la celda del objetivo {target.name} en {target.GridPosition}");
            return;
        }

        Debug.Log($"[CombatManager] Intentando disparar a {target.name} en {target.GridPosition}");

        bool isShootable = ContainsCellWithCoordinates(currentShootableCells, targetCell.Coordinates);

        if (!isShootable)
        {
            Debug.LogWarning($"[CombatManager] El objetivo en {targetCell.Coordinates} no est� en la lista de celdas atacables.");
            return;
        }

        StartCoroutine(ShootRoutine(target));
    }

    private IEnumerator ShootRoutine(UnitController target)
    {
        isBusy = true;

        yield return new WaitForSeconds(0.25f);

        int damage = CalculateDamage(selectedUnit, target);
        target.TakeDamage(damage);

        selectedUnit.SpendActionPoint();
        currentMode = ActionMode.None;
        currentReachableCells.Clear();
        currentShootableCells.Clear();

        RefreshSelectionVisuals();

        if (target == null || target.CurrentHealth <= 0)
        {
            CombatFinished = true;
        }

        isBusy = false;
    }

    private int CalculateDamage(UnitController attacker, UnitController target)
    {
        CoverType cover = gridManager.GetCoverAgainstAttacker(attacker, target);

        switch (cover)
        {
            case CoverType.Full:
                return 0;

            case CoverType.Half:
                return 1;

            default:
                return 2;
        }
    }

    private void EndPlayerTurn()
    {
        ClearSelection();
        turnSystem.EndTurn();
        StartCoroutine(EnemyTurnRoutine());
    }

    private void ClearSelection()
    {
        if (selectedUnit != null)
            selectedUnit.Deselect();

        selectedUnit = null;
        currentMode = ActionMode.None;
        currentReachableCells.Clear();
        currentShootableCells.Clear();

        if (gridManager != null)
            gridManager.ResetAllCellVisuals();

        RefreshHUD();
    }

    private IEnumerator EnemyTurnRoutine()
    {
        isBusy = true;

        if (enemyUnit != null)
        {
            enemyUnit.StartTurn();
            yield return new WaitForSeconds(0.75f);
        }

        if (playerUnit != null)
            playerUnit.StartTurn();

        turnSystem.EndTurn();
        isBusy = false;
    }

    private void RefreshHUD()
    {
        if (combatHUD == null)
            return;

        CombatHUD.HUDMode hudMode = CombatHUD.HUDMode.None;

        switch (currentMode)
        {
            case ActionMode.Move:
                hudMode = CombatHUD.HUDMode.Move;
                break;

            case ActionMode.Shoot:
                hudMode = CombatHUD.HUDMode.Shoot;
                break;
        }

        bool hasSelected = selectedUnit != null;
        bool canAct = hasSelected && selectedUnit.HasActionPoints();

        combatHUD.Refresh(hasSelected, canAct, hudMode);
    }

    private bool ContainsCellWithCoordinates(List<GridCell> cells, Vector2Int coordinates)
    {
        for (int i = 0; i < cells.Count; i++)
        {
            GridCell cell = cells[i];

            if (cell == null)
                continue;

            if (cell.Coordinates == coordinates)
                return true;
        }

        return false;
    }
}
```

=== CombatTile.cs ===
```csharp
using UnityEngine;

[ExecuteAlways]
public class CombatTile : MonoBehaviour
{
    [SerializeField] private Vector2Int coordinates;
    [SerializeField] private bool walkable = true;

    [Header("Directional Cover")]
    [SerializeField] private CoverType upCover = CoverType.None;
    [SerializeField] private CoverType downCover = CoverType.None;
    [SerializeField] private CoverType leftCover = CoverType.None;
    [SerializeField] private CoverType rightCover = CoverType.None;

    public Vector2Int Coordinates => coordinates;
    public bool Walkable => walkable;
    public CoverType UpCover => upCover;
    public CoverType DownCover => downCover;
    public CoverType LeftCover => leftCover;
    public CoverType RightCover => rightCover;

    private void OnValidate()
    {
        // Sincroniza coordinates con la posici�n del transform y actualiza el nombre del GameObject
        SyncCoordinatesWithTransform();
    }

    /// <summary>
    /// Igualar coordinates.x con transform.position.x y coordinates.y con transform.position.z.
    /// Se usa RoundToInt para convertir de float a int de forma estable.
    /// </summary>
    public void SyncCoordinatesWithTransform()
    {
        Vector3 pos = transform.position;
        coordinates.x = Mathf.RoundToInt(pos.x);
        coordinates.y = Mathf.RoundToInt(pos.z);

        gameObject.name = $"Tile_{coordinates.x}_{coordinates.y}";

#if UNITY_EDITOR
        // Marca el objeto como modificado en el editor para que Unity persista cambios serializados
        UnityEditor.EditorUtility.SetDirty(this);
#endif
    }
}
```

=== CombatZone.cs ===
```csharp
using UnityEngine;

public class CombatZone : MonoBehaviour
{
    [SerializeField] private GridManager gridManager;
    [SerializeField] private Transform tilesRoot;
    [SerializeField] private Transform playerUnitsRoot;
    [SerializeField] private Transform enemyUnitsRoot;

    public GridManager GridManager => gridManager;
    public Transform TilesRoot => tilesRoot;
    public Transform PlayerUnitsRoot => playerUnitsRoot;
    public Transform EnemyUnitsRoot => enemyUnitsRoot;

    private void OnTriggerEnter(Collider other)
    {
        if (!other.CompareTag("Player"))
            return;

        PlayerStateMachine playerStateMachine = other.GetComponent<PlayerStateMachine>();
        if (playerStateMachine == null)
            return;

        CombatManager combatManager = playerStateMachine.GetCombatManager();
        combatManager.SetCombatZone(this);
        playerStateMachine.ChangeState(playerStateMachine.CombatModeState);
    }
}
```

=== CombatZoneMember.cs ===
```csharp
using UnityEngine;

public class CombatZoneMember : MonoBehaviour
{
    [SerializeField] private Vector2Int startingCoordinates;

    public Vector2Int StartingCoordinates => startingCoordinates;

    private void OnValidate()
    {
        SyncStartingCoordinatesWithTransform();
    }

    /// <summary>
    /// Igualar startingCoordinates.x con transform.position.x y startingCoordinates.y con transform.position.z.
    /// Se usa Mathf.RoundToInt para convertir de float a int de forma estable.
    /// </summary>
    public void SyncStartingCoordinatesWithTransform()
    {
        Vector3 pos = transform.position;
        startingCoordinates.x = Mathf.RoundToInt(pos.x);
        startingCoordinates.y = Mathf.RoundToInt(pos.z);

#if UNITY_EDITOR
        // Marca el objeto como modificado en el editor para que Unity persista cambios serializados
        UnityEditor.EditorUtility.SetDirty(this);
#endif
    }
}
```

=== TurnSystem.cs ===
```csharp
using UnityEngine;

public class TurnSystem : MonoBehaviour
{
    public enum TeamTurn
    {
        Player,
        Enemy
    }

    public TeamTurn CurrentTurn { get; private set; }

    public void StartTurns()
    {
        CurrentTurn = TeamTurn.Player;
        Debug.Log("Empieza turno del jugador");
    }

    public void EndTurn()
    {
        if (CurrentTurn == TeamTurn.Player)
        {
            CurrentTurn = TeamTurn.Enemy;
            Debug.Log("Empieza turno del enemigo");
        }
        else
        {
            CurrentTurn = TeamTurn.Player;
            Debug.Log("Empieza turno del jugador");
        }
    }

    public bool IsPlayerTurn()
    {
        return CurrentTurn == TeamTurn.Player;
    }
}
```

=== CellVisual.cs ===
```csharp
using UnityEngine;

[ExecuteAlways]
public class CellVisual : MonoBehaviour
{
    [Header("Configuración del Tile (Editor)")]
    [SerializeField] private GameObject tilePrefab;

    [Header("Colores de Estado")]
    [SerializeField] private Color normalColor = Color.white;
    [SerializeField] private Color reachableColor = Color.cyan;
    [SerializeField] private Color selectedColor = Color.yellow;
    [SerializeField] private Color attackableColor = Color.red;

    [HideInInspector][SerializeField] private GameObject instanciaHijoVisual;

    private Renderer childRenderer;
    private MaterialPropertyBlock propertyBlock;
    private Color currentColor;

    private static readonly int BaseColorPropertyID = Shader.PropertyToID("_BaseColor");
    private static readonly int LegacyColorPropertyID = Shader.PropertyToID("_Color");

    private void Awake()
    {
        propertyBlock = new MaterialPropertyBlock();
        currentColor = normalColor;

        GestionarComponentesRaiz();

        if (Application.isPlaying)
        {
            if (gameObject.scene.rootCount == 0) return;
            ActualizarReferenciasHijo();
            SetNormal();
        }
    }

    // Al añadir este atributo, creas una función que puedes activar manualmente desde el Inspector
    [ContextMenu("✓ Actualizar Estructura del Tile")]
    public void ActualizarInstanciaTile()
    {
        // REGLA 1: Si no hay prefab asignado, limpiamos el hijo y encendemos la raíz (SIN MODIFICAR EL TRANSFORM)
        if (tilePrefab == null)
        {
            LimpiarHijoViejo();
            RestaurarComponentesOriginalesRaiz();
            return;
        }

        // Si ya tiene el hijo correcto puesto, aseguramos los componentes y salimos para evitar bucles
        if (instanciaHijoVisual != null && instanciaHijoVisual.name == "Visual_" + tilePrefab.name)
        {
            GestionarComponentesRaiz();
            return;
        }

        LimpiarHijoViejo();

        // REGLA 2: Instanciamos el hijo manteniendo de forma estricta todos sus números originales
#if UNITY_EDITOR
        if (UnityEditor.PrefabUtility.IsPartOfPrefabAsset(tilePrefab))
            instanciaHijoVisual = (GameObject)UnityEditor.PrefabUtility.InstantiatePrefab(tilePrefab, transform);
        else
            instanciaHijoVisual = Instantiate(tilePrefab, transform);
#else
        instanciaHijoVisual = Instantiate(tilePrefab, transform);
#endif

        if (instanciaHijoVisual != null)
        {
            instanciaHijoVisual.transform.localPosition = tilePrefab.transform.localPosition;
            instanciaHijoVisual.transform.localRotation = tilePrefab.transform.localRotation;
            instanciaHijoVisual.transform.localScale = tilePrefab.transform.localScale;
            instanciaHijoVisual.name = "Visual_" + tilePrefab.name;

#if UNITY_EDITOR
            UnityEditor.Undo.RegisterCreatedObjectUndo(instanciaHijoVisual, "Asignar Tile Prefab");
#endif
        }

        // REGLA 3: Pone al padre en escala 2,2,2 y apaga sus componentes visuales
        GestionarComponentesRaiz();
        ActualizarReferenciasHijo();

#if UNITY_EDITOR
        UnityEditor.EditorUtility.SetDirty(gameObject);
        UnityEditor.SceneView.RepaintAll();
#endif
    }

    private void GestionarComponentesRaiz()
    {
        if (tilePrefab != null)
        {
            // Forzamos la escala (2,2,2) en el GameObject original únicamente si hay un prefab puesto
            if (transform.localScale != new Vector3(2f, 2f, 2f))
            {
                transform.localScale = new Vector3(2f, 2f, 2f);
            }

            var rootRenderer = GetComponent<Renderer>();
            var rootCollider = GetComponent<Collider>();

            if (rootRenderer != null) rootRenderer.enabled = false;
            if (rootCollider != null) rootCollider.enabled = false;
        }
    }

    private void RestaurarComponentesOriginalesRaiz()
    {
        // Volvemos a encender la malla y colisionador base de la celda sin alterar la escala del padre
        var rootRenderer = GetComponent<Renderer>();
        var rootCollider = GetComponent<Collider>();

        if (rootRenderer != null) rootRenderer.enabled = true;
        if (rootCollider != null) rootCollider.enabled = true;
    }

    private void LimpiarHijoViejo()
    {
        for (int i = transform.childCount - 1; i >= 0; i--)
        {
            GameObject hijo = transform.GetChild(i).gameObject;
            if (hijo != null && hijo.name.StartsWith("Visual_"))
            {
#if UNITY_EDITOR
                UnityEditor.Undo.DestroyObjectImmediate(hijo);
#else
                Destroy(hijo);
#endif
            }
        }
        instanciaHijoVisual = null;
    }

    private void ActualizarReferenciasHijo()
    {
        if (instanciaHijoVisual != null)
        {
            childRenderer = instanciaHijoVisual.GetComponentInChildren<Renderer>();
        }
    }

    private void ChangeColor(Color color)
    {
        if (!Application.isPlaying) return;

        currentColor = color;
        if (childRenderer == null) ActualizarReferenciasHijo();
        if (childRenderer == null) return;

        if (propertyBlock == null) propertyBlock = new MaterialPropertyBlock();

        childRenderer.GetPropertyBlock(propertyBlock);
        propertyBlock.SetColor(BaseColorPropertyID, color);
        propertyBlock.SetColor(LegacyColorPropertyID, color);
        childRenderer.SetPropertyBlock(propertyBlock);
    }

    public void SetNormal() => ChangeColor(normalColor);
    public void SetReachable() => ChangeColor(reachableColor);
    public void SetSelected() => ChangeColor(selectedColor);
    public void SetAttackable() => ChangeColor(attackableColor);
}

```

=== CoverDirection.cs ===
```csharp
public enum CoverDirection
{
    Up,
    Down,
    Left,
    Right
}
```

=== CoverType.cs ===
```csharp
public enum CoverType
{
    None,
    Half,
    Full
}
```

=== GridCell.cs ===
```csharp
using UnityEngine;

public class GridCell : MonoBehaviour
{
    public Vector2Int Coordinates { get; private set; }
    public bool Walkable { get; private set; } = true;
    public UnitController Occupant { get; private set; }

    [Header("Directional Cover")]
    [SerializeField] private CoverType upCover = CoverType.None;
    [SerializeField] private CoverType downCover = CoverType.None;
    [SerializeField] private CoverType leftCover = CoverType.None;
    [SerializeField] private CoverType rightCover = CoverType.None;

    private CellVisual cellVisual;

    private void Awake()
    {
        cellVisual = GetComponent<CellVisual>();
    }

    public void Initialize(Vector2Int coordinates)
    {
        Coordinates = coordinates;
        gameObject.name = $"Cell_{coordinates.x}_{coordinates.y}";
        SetNormalVisual();
    }

    public bool IsOccupied()
    {
        return Occupant != null;
    }

    public void SetOccupant(UnitController unit)
    {
        Occupant = unit;
    }

    public void ClearOccupant()
    {
        Occupant = null;
    }

    public void SetWalkable(bool value)
    {
        Walkable = value;
    }

    public void SetCover(CoverDirection direction, CoverType coverType)
    {
        switch (direction)
        {
            case CoverDirection.Up:
                upCover = coverType;
                break;
            case CoverDirection.Down:
                downCover = coverType;
                break;
            case CoverDirection.Left:
                leftCover = coverType;
                break;
            case CoverDirection.Right:
                rightCover = coverType;
                break;
        }
    }

    public CoverType GetCoverFromDirection(CoverDirection direction)
    {
        return direction switch
        {
            CoverDirection.Up => upCover,
            CoverDirection.Down => downCover,
            CoverDirection.Left => leftCover,
            CoverDirection.Right => rightCover,
            _ => CoverType.None
        };
    }

    public void SetNormalVisual()
    {
        cellVisual?.SetNormal();
    }

    public void SetReachableVisual()
    {
        cellVisual?.SetReachable();
    }

    public void SetSelectedVisual()
    {
        cellVisual?.SetSelected();
    }

    public void SetAttackableVisual()
    {
        cellVisual?.SetAttackable();
    }
}
```

=== GridManager.cs ===
```csharp
using UnityEngine;
using System.Collections.Generic;

public class GridManager : MonoBehaviour
{
    [Header("Configuraci�n de la Grid")]
    [SerializeField] private int gridStep = 2;

    private Dictionary<Vector2Int, GridCell> grid = new Dictionary<Vector2Int, GridCell>();

    public void BuildGridFromScene(Transform tilesRoot)
    {
        grid.Clear();

        if (tilesRoot == null)
        {
            Debug.LogError("[GridManager] tilesRoot es null.");
            return;
        }

        CombatTile[] combatTiles = tilesRoot.GetComponentsInChildren<CombatTile>(true);

        Debug.Log($"[GridManager] BuildGridFromScene: encontrados {combatTiles.Length} CombatTile(s).");

        foreach (CombatTile combatTile in combatTiles)
        {
            if (combatTile == null)
                continue;

            GridCell gridCell = combatTile.GetComponent<GridCell>();

            if (gridCell == null)
            {
                Debug.LogWarning($"[GridManager] El objeto {combatTile.name} tiene CombatTile pero no GridCell.");
                continue;
            }

            gridCell.Initialize(combatTile.Coordinates);
            gridCell.SetWalkable(combatTile.Walkable);

            gridCell.SetCover(CoverDirection.Up, combatTile.UpCover);
            gridCell.SetCover(CoverDirection.Down, combatTile.DownCover);
            gridCell.SetCover(CoverDirection.Left, combatTile.LeftCover);
            gridCell.SetCover(CoverDirection.Right, combatTile.RightCover);

            if (grid.ContainsKey(combatTile.Coordinates))
            {
                Debug.LogWarning($"[GridManager] Coordenada duplicada detectada: {combatTile.Coordinates}. Se sobrescribir� con {combatTile.name}");
            }

            grid[combatTile.Coordinates] = gridCell;

            Debug.Log($"[GridManager] Registrada celda {combatTile.Coordinates} | walkable={combatTile.Walkable} | world={gridCell.transform.position}");
        }

        Debug.Log($"[GridManager] Grid construida con {grid.Count} celdas. gridStep={gridStep}");
    }

    public GridCell GetCell(Vector2Int coords)
    {
        if (grid.TryGetValue(coords, out GridCell cell))
            return cell;

        return null;
    }

    public Vector3 GetWorldPosition(Vector2Int coords)
    {
        GridCell cell = GetCell(coords);
        return cell != null ? cell.transform.position : Vector3.zero;
    }

    public Vector3 GetCellCenterWorld(GridCell cell)
    {
        return cell.transform.position + Vector3.up * 0.5f;
    }

    public List<GridCell> GetNeighbours(GridCell cell)
    {
        List<GridCell> neighbours = new List<GridCell>();

        if (cell == null)
        {
            Debug.LogWarning("[GridManager] GetNeighbours recibi� una celda null.");
            return neighbours;
        }

        Vector2Int[] directions =
        {
            new Vector2Int(gridStep, 0),
            new Vector2Int(-gridStep, 0),
            new Vector2Int(0, gridStep),
            new Vector2Int(0, -gridStep)
        };

        foreach (Vector2Int dir in directions)
        {
            Vector2Int neighbourCoords = cell.Coordinates + dir;
            GridCell neighbour = GetCell(neighbourCoords);

            if (neighbour != null)
            {
                neighbours.Add(neighbour);
            }
        }

        return neighbours;
    }

    public List<GridCell> GetReachableCells(UnitController unit)
    {
        List<GridCell> reachableCells = new List<GridCell>();

        if (unit == null)
        {
            Debug.LogError("[GridManager] GetReachableCells recibi� unit null.");
            return reachableCells;
        }

        Debug.Log($"[GridManager] GetReachableCells para {unit.name} desde {unit.GridPosition} con MoveRange={unit.MoveRange}");

        Queue<(GridCell cell, int distance)> queue = new Queue<(GridCell, int)>();
        HashSet<GridCell> visited = new HashSet<GridCell>();

        GridCell startCell = GetCell(unit.GridPosition);

        if (startCell == null)
        {
            Debug.LogError($"[GridManager] No existe startCell para la posici�n {unit.GridPosition}");
            return reachableCells;
        }

        Debug.Log($"[GridManager] startCell encontrada: {startCell.name} en {startCell.Coordinates}, walkable={startCell.Walkable}, occupied={startCell.IsOccupied()}");

        queue.Enqueue((startCell, 0));
        visited.Add(startCell);

        while (queue.Count > 0)
        {
            var current = queue.Dequeue();
            GridCell currentCell = current.cell;
            int distance = current.distance;

            Debug.Log($"[GridManager] Visitando {currentCell.Coordinates} a distancia {distance}");

            if (distance > 0)
            {
                reachableCells.Add(currentCell);
            }

            if (distance >= unit.MoveRange)
            {
                Debug.Log($"[GridManager] Se alcanza el rango m�ximo en {currentCell.Coordinates}");
                continue;
            }

            List<GridCell> neighbours = GetNeighbours(currentCell);

            foreach (GridCell neighbour in neighbours)
            {
                if (visited.Contains(neighbour))
                {
                    Debug.Log($"[GridManager] Vecino {neighbour.Coordinates} descartado: ya visitado");
                    continue;
                }

                if (!neighbour.Walkable)
                {
                    Debug.Log($"[GridManager] Vecino {neighbour.Coordinates} descartado: no walkable");
                    continue;
                }

                bool occupiedByOtherUnit = neighbour.IsOccupied() && neighbour.Occupant != unit;
                if (occupiedByOtherUnit)
                {
                    Debug.Log($"[GridManager] Vecino {neighbour.Coordinates} descartado: ocupado por {neighbour.Occupant.name}");
                    continue;
                }

                Debug.Log($"[GridManager] Vecino {neighbour.Coordinates} a�adido a la cola con distancia {distance + 1}");

                visited.Add(neighbour);
                queue.Enqueue((neighbour, distance + 1));
            }
        }

        Debug.Log($"[GridManager] Total reachableCells: {reachableCells.Count}");

        return reachableCells;
    }

    public List<GridCell> GetShootableCells(UnitController shooter)
    {
        List<GridCell> shootableCells = new List<GridCell>();

        if (shooter == null)
        {
            Debug.LogError("[GridManager] GetShootableCells recibi� shooter null.");
            return shootableCells;
        }

        Debug.Log($"[GridManager] GetShootableCells para {shooter.name} desde {shooter.GridPosition} con ShootRange={shooter.ShootRange}");

        foreach (GridCell cell in grid.Values)
        {
            if (cell == null || !cell.IsOccupied())
                continue;

            UnitController target = cell.Occupant;

            if (target == null)
                continue;

            if (target.IsEnemy == shooter.IsEnemy)
                continue;

            int distance =
                Mathf.Abs(cell.Coordinates.x - shooter.GridPosition.x) / gridStep +
                Mathf.Abs(cell.Coordinates.y - shooter.GridPosition.y) / gridStep;

            if (distance > shooter.ShootRange)
                continue;

            shootableCells.Add(cell);
            Debug.Log($"[GridManager] Celda atacable -> {cell.Coordinates} con objetivo {target.name}");
        }

        Debug.Log($"[GridManager] Total shootableCells: {shootableCells.Count}");

        return shootableCells;
    }

    public CoverType GetCoverAgainstAttacker(UnitController attacker, UnitController target)
    {
        GridCell targetCell = GetCell(target.GridPosition);
        if (targetCell == null)
            return CoverType.None;

        int dx = attacker.GridPosition.x - target.GridPosition.x;
        int dy = attacker.GridPosition.y - target.GridPosition.y;

        if (Mathf.Abs(dx) > Mathf.Abs(dy))
        {
            if (dx > 0)
                return targetCell.GetCoverFromDirection(CoverDirection.Right);
            else
                return targetCell.GetCoverFromDirection(CoverDirection.Left);
        }
        else
        {
            if (dy > 0)
                return targetCell.GetCoverFromDirection(CoverDirection.Up);
            else
                return targetCell.GetCoverFromDirection(CoverDirection.Down);
        }
    }

    public void ResetAllCellVisuals()
    {
        foreach (GridCell cell in grid.Values)
        {
            if (cell != null)
                cell.SetNormalVisual();
        }
    }

    public void HighlightCells(List<GridCell> cells)
    {
        foreach (GridCell cell in cells)
        {
            if (cell != null)
                cell.SetReachableVisual();
        }
    }

    public void HighlightAttackableCells(List<GridCell> cells)
    {
        foreach (GridCell cell in cells)
        {
            if (cell != null)
                cell.SetAttackableVisual();
        }
    }
}
```

=== IPlayerState.cs ===
```csharp
public interface IPlayerState
{
    void Enter();
    void Tick();
    void Exit();
}
```

=== PlayerCombatModeState.cs ===
```csharp
using UnityEngine;

public class PlayerCombatModeState : IPlayerState
{
    private readonly PlayerStateMachine stateMachine;
    private readonly CombatManager combatManager;

    public PlayerCombatModeState(PlayerStateMachine stateMachine, CombatManager combatManager)
    {
        this.stateMachine = stateMachine;
        this.combatManager = combatManager;
    }

    public void Enter()
    {
        Debug.Log("Entrando en modo combate");
        combatManager.BeginCombat();
    }

    public void Tick()
    {
        if (combatManager.CombatFinished)
        {
            stateMachine.ChangeState(stateMachine.ExplorationState);
        }
    }

    public void Exit()
    {
        Debug.Log("Saliendo de modo combate");
        combatManager.EndCombat();
    }
}
```

=== PlayerExplorationState.cs ===
```csharp
public class PlayerExplorationState : IPlayerState
{
    private readonly PlayerStateMachine stateMachine;

    public PlayerExplorationState(PlayerStateMachine stateMachine)
    {
        this.stateMachine = stateMachine;
    }

    public void Enter()
    {
    }

    public void Tick()
    {
    }

    public void Exit()
    {
    }
}
```

=== PlayerStateMachine.cs ===
```csharp
using UnityEngine;

public class PlayerStateMachine : MonoBehaviour
{
    private IPlayerState currentState;

    public PlayerExplorationState ExplorationState { get; private set; }
    public PlayerCombatModeState CombatModeState { get; private set; }

    [SerializeField] private CombatManager combatManager;

    private void Awake()
    {
        ExplorationState = new PlayerExplorationState(this);
        CombatModeState = new PlayerCombatModeState(this, combatManager);
    }

    private void Start()
    {
        ChangeState(ExplorationState);
    }

    private void Update()
    {
        currentState?.Tick();
    }

    public void ChangeState(IPlayerState newState)
    {
        currentState?.Exit();
        currentState = newState;
        currentState?.Enter();
    }

    public CombatManager GetCombatManager()
    {
        return combatManager;
    }
}

```

=== UnitController.cs ===
```csharp
using UnityEngine;
using System.Collections;

public class UnitController : MonoBehaviour
{
    [SerializeField] private int moveRange = 4;
    [SerializeField] private int shootRange = 5;
    [SerializeField] private int maxHealth = 3;
    [SerializeField] private bool isEnemy = false;

    public Vector2Int GridPosition { get; private set; }
    public int MoveRange => moveRange;
    public int ShootRange => shootRange;
    public bool IsEnemy => isEnemy;
    public bool IsSelected { get; private set; }
    public int CurrentHealth { get; private set; }
    public int ActionPoints { get; private set; }

    private void Awake()
    {
        CurrentHealth = maxHealth;
        ActionPoints = 2;
    }

    public void StartTurn()
    {
        ActionPoints = 2;
    }

    public bool HasActionPoints()
    {
        return ActionPoints > 0;
    }

    public void SpendActionPoint()
    {
        ActionPoints = Mathf.Max(0, ActionPoints - 1);
    }

    public void Place(GridCell cell, GridManager gridManager)
    {
        if (cell == null)
        {
            Debug.LogError($"[UnitController] No se puede colocar {name}: cell es null.");
            return;
        }

        if (gridManager == null)
        {
            Debug.LogError($"[UnitController] No se puede colocar {name}: gridManager es null.");
            return;
        }

        GridCell previousCell = gridManager.GetCell(GridPosition);
        if (previousCell != null && previousCell.Occupant == this)
        {
            previousCell.ClearOccupant();
        }

        GridPosition = cell.Coordinates;
        transform.position = gridManager.GetWorldPosition(GridPosition) + Vector3.up * 0.5f;
        cell.SetOccupant(this);

        Debug.Log($"[UnitController] {name} colocado en GridPosition={GridPosition}, world={transform.position}");
    }

    public IEnumerator MoveTo(GridCell fromCell, GridCell toCell, GridManager gridManager)
    {
        if (fromCell == null)
        {
            Debug.LogError($"[UnitController] {name} no puede moverse: fromCell es null.");
            yield break;
        }

        if (toCell == null)
        {
            Debug.LogError($"[UnitController] {name} no puede moverse: toCell es null.");
            yield break;
        }

        if (gridManager == null)
        {
            Debug.LogError($"[UnitController] {name} no puede moverse: gridManager es null.");
            yield break;
        }

        fromCell.ClearOccupant();
        toCell.SetOccupant(this);

        Vector3 start = transform.position;
        Vector3 end = gridManager.GetWorldPosition(toCell.Coordinates) + Vector3.up * 0.5f;

        float duration = 0.25f;
        float t = 0f;

        while (t < duration)
        {
            t += Time.deltaTime;
            transform.position = Vector3.Lerp(start, end, t / duration);
            yield return null;
        }

        transform.position = end;
        GridPosition = toCell.Coordinates;

        Debug.Log($"[UnitController] {name} movido a GridPosition={GridPosition}, world={transform.position}");
    }

    public void Select()
    {
        IsSelected = true;
    }

    public void Deselect()
    {
        IsSelected = false;
    }

    public void TakeDamage(int damage)
    {
        CurrentHealth -= damage;

        Debug.Log($"[UnitController] {name} recibe {damage} de da�o. Vida restante: {CurrentHealth}");

        if (CurrentHealth <= 0)
        {
            Debug.Log($"[UnitController] {name} ha muerto.");
            Destroy(gameObject);
        }
    }
}
```

