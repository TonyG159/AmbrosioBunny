# Bunny Game All Scripts

- Proyecto: Bunny Game
- Carpeta origen: `Assets/Project`
- Total scripts: 16
- Exportado: 2026-07-19 14:31:07

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

    [Header("Main Labels")]
    [SerializeField] private TMP_Text moveLabel;
    [SerializeField] private TMP_Text shootLabel;
    [SerializeField] private TMP_Text cancelLabel;
    [SerializeField] private TMP_Text endTurnLabel;
    [SerializeField] private TMP_Text statusText;

    [Header("Info Labels")]
    [SerializeField] private TMP_Text selectedUnitText;
    [SerializeField] private TMP_Text actionPointsText;
    [SerializeField] private TMP_Text coverText;
    [SerializeField] private TMP_Text targetText;

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
            SetStatus("Selecciona una unidad");
            SetActionPoints("-");
            SetSelectedUnit("-");
            SetTarget("-");
            SetCover("-");
            SetPanelState(movePanel, false);
            SetPanelState(shootPanel, false);
            SetPanelState(cancelPanel, false);
            SetPanelState(endTurnPanel, true);
            return;
        }

        if (!canAct)
        {
            SetStatus("Unidad sin acciones");
            SetPanelState(movePanel, false);
            SetPanelState(shootPanel, false);
            SetPanelState(cancelPanel, false);
            SetPanelState(endTurnPanel, true);
            return;
        }

        SetPanelState(movePanel, true);
        SetPanelState(shootPanel, true);
        SetPanelState(cancelPanel, true);
        SetPanelState(endTurnPanel, true);

        switch (mode)
        {
            case HUDMode.Move:
                SetStatus("Modo mover activo");
                HighlightPanel(movePanel);
                break;

            case HUDMode.Shoot:
                SetStatus("Modo disparo activo");
                HighlightPanel(shootPanel);
                break;

            case HUDMode.None:
                SetStatus("Esperando acci n");
                break;
        }
    }

    public void SetSelectedUnit(string value)
    {
        if (selectedUnitText != null)
            selectedUnitText.text = $"Unidad: {value}";
    }

    public void SetActionPoints(string value)
    {
        if (actionPointsText != null)
            actionPointsText.text = $"AP: {value}";
    }

    public void SetCover(string value)
    {
        if (coverText != null)
            coverText.text = $"Cobertura: {value}";
    }

    public void SetTarget(string value)
    {
        if (targetText != null)
            targetText.text = $"Objetivo: {value}";
    }

    public void SetStatus(string value)
    {
        if (statusText != null)
            statusText.text = value;
    }

    private void ResetPanels()
    {
        if (movePanel != null) movePanel.color = inactiveColor;
        if (shootPanel != null) shootPanel.color = inactiveColor;
        if (cancelPanel != null) cancelPanel.color = inactiveColor;
        if (endTurnPanel != null) endTurnPanel.color = inactiveColor;
    }

    private void HighlightPanel(Image panel)
    {
        if (panel != null)
            panel.color = activeColor;
    }

    private void SetPanelState(Image panel, bool enabled)
    {
        if (panel != null)
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

    [Header("References")]
    [SerializeField] private GridManager gridManager;
    [SerializeField] private TurnSystem turnSystem;
    [SerializeField] private CombatHUD combatHUD;

    [Header("Raycast Layers")]
    [SerializeField] private LayerMask unitLayerMask;
    [SerializeField] private LayerMask gridCellLayerMask;

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
            Debug.LogError("[CombatManager] TurnSystem no est  asignado.");
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
                GridCell playerSpawn = gridManager.GetCell(playerMember.StartingCoordinates);
                if (playerSpawn != null)
                {
                    playerUnit.Place(playerSpawn, gridManager);
                    playerUnit.StartTurn();
                }
                else
                {
                    Debug.LogError($"[CombatManager] No existe GridCell para el jugador en {playerMember.StartingCoordinates}");
                }
            }
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
                GridCell enemySpawn = gridManager.GetCell(enemyMember.StartingCoordinates);
                if (enemySpawn != null)
                {
                    enemyUnit.Place(enemySpawn, gridManager);
                    enemyUnit.StartTurn();
                }
                else
                {
                    Debug.LogError($"[CombatManager] No existe GridCell para el enemigo en {enemyMember.StartingCoordinates}");
                }
            }
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

        if (Physics.Raycast(ray, out RaycastHit unitHit, 100f, unitLayerMask, QueryTriggerInteraction.Ignore))
        {
            UnitController clickedUnit = unitHit.collider.GetComponentInParent<UnitController>();
            if (clickedUnit != null)
            {
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
        }

        if (Physics.Raycast(ray, out RaycastHit cellHit, 100f, gridCellLayerMask, QueryTriggerInteraction.Ignore))
        {
            GridCell clickedCell = cellHit.collider.GetComponentInParent<GridCell>();
            if (clickedCell != null)
            {
                if (currentMode == ActionMode.Move)
                {
                    TryMoveSelectedUnit(clickedCell);
                    return;
                }
            }
        }
    }

    private void SelectUnit(UnitController unit)
    {
        if (unit == null)
            return;

        ClearSelection();

        selectedUnit = unit;
        selectedUnit.Select();
        currentMode = ActionMode.None;

        RefreshSelectionVisuals();
        RefreshHUD();
    }

    private void EnterMoveMode()
    {
        if (selectedUnit == null)
            return;

        currentMode = ActionMode.Move;
        currentReachableCells = gridManager.GetReachableCells(selectedUnit);
        currentShootableCells.Clear();

        RefreshSelectionVisuals();
        RefreshHUD();
    }

    private void EnterShootMode()
    {
        if (selectedUnit == null)
            return;

        currentMode = ActionMode.Shoot;
        currentShootableCells = gridManager.GetShootableCells(selectedUnit);
        currentReachableCells.Clear();

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
        if (selectedUnit == null || targetCell == null)
            return;

        GridCell currentCell = gridManager.GetCell(selectedUnit.GridPosition);
        if (currentCell == null)
            return;

        List<GridCell> path = gridManager.FindPath(selectedUnit, currentCell, targetCell);

        if (path == null || path.Count < 2)
        {
            Debug.LogWarning("[CombatManager] No existe una ruta v lida hasta esa celda.");
            return;
        }

        StartCoroutine(MoveSelectedUnitRoutine(path));
    }

    private IEnumerator MoveSelectedUnitRoutine(List<GridCell> path)
    {
        isBusy = true;

        yield return selectedUnit.MoveAlongPath(path, gridManager);

        selectedUnit.SpendActionPoint();
        currentMode = ActionMode.None;
        currentReachableCells.Clear();
        currentShootableCells.Clear();

        RefreshSelectionVisuals();

        isBusy = false;
    }

    private void TryShootEnemy(UnitController target)
    {
        if (selectedUnit == null || target == null)
            return;

        GridCell targetCell = gridManager.GetCell(target.GridPosition);

        if (targetCell == null)
            return;

        bool isShootable = ContainsCellWithCoordinates(currentShootableCells, targetCell.Coordinates);

        if (!isShootable)
        {
            Debug.LogWarning("[CombatManager] El objetivo no est  en una celda atacable.");
            return;
        }

        StartCoroutine(ShootRoutine(target));
    }

    private IEnumerator ShootRoutine(UnitController target)
    {
        isBusy = true;

        yield return new WaitForSeconds(0.25f);

        CoverType cover = gridManager.GetCoverAgainstAttacker(selectedUnit, target);
        int damage = CalculateDamage(selectedUnit, target, cover);

        Debug.Log($"[CombatManager] Disparo contra {target.name}. Resultado de cobertura: {gridManager.GetCoverDescriptionAgainstAttacker(selectedUnit, target)}. Da o: {damage}");

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

    private int CalculateDamage(UnitController attacker, UnitController target, CoverType cover)
    {
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
        // Sincroniza coordinates con la posici n del transform y actualiza el nombre del GameObject
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

    private static readonly int BaseColorPropertyID = Shader.PropertyToID("_BaseColor");
    private static readonly int LegacyColorPropertyID = Shader.PropertyToID("_Color");

    private void Awake()
    {
        propertyBlock = new MaterialPropertyBlock();

        GestionarComponentesRaiz();

        if (Application.isPlaying)
        {
            if (gameObject.scene.rootCount == 0)
                return;

            ResolveRenderer();
            SetNormal();
        }
    }

    [ContextMenu("✓ Actualizar Estructura del Tile")]
    public void ActualizarInstanciaTile()
    {
        if (tilePrefab == null)
        {
            LimpiarHijoViejo();
            RestaurarComponentesOriginalesRaiz();
            return;
        }

        if (instanciaHijoVisual != null && instanciaHijoVisual.name == "Visual_" + tilePrefab.name)
        {
            GestionarComponentesRaiz();
            ResolveRenderer();
            return;
        }

        LimpiarHijoViejo();

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

        GestionarComponentesRaiz();
        ResolveRenderer();

#if UNITY_EDITOR
        UnityEditor.EditorUtility.SetDirty(gameObject);
        UnityEditor.SceneView.RepaintAll();
#endif
    }

    private void GestionarComponentesRaiz()
    {
        if (tilePrefab != null)
        {
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
        childRenderer = null;
    }

    private void ResolveRenderer()
    {
        childRenderer = null;

        if (instanciaHijoVisual != null)
        {
            childRenderer = instanciaHijoVisual.GetComponentInChildren<Renderer>(true);
        }

        if (childRenderer == null)
        {
            Renderer[] renderers = GetComponentsInChildren<Renderer>(true);
            foreach (Renderer r in renderers)
            {
                if (r != null && r.gameObject != gameObject)
                {
                    childRenderer = r;
                    break;
                }
            }
        }

        if (childRenderer == null)
        {
            childRenderer = GetComponent<Renderer>();
        }

        if (childRenderer == null)
        {
            Debug.LogWarning($"[CellVisual] {name}: no se encontró ningún Renderer para aplicar colores.");
        }
    }

    private void ChangeColor(Color color)
    {
        if (!Application.isPlaying)
            return;

        if (childRenderer == null)
            ResolveRenderer();

        if (childRenderer == null)
            return;

        if (propertyBlock == null)
            propertyBlock = new MaterialPropertyBlock();

        childRenderer.GetPropertyBlock(propertyBlock);
        propertyBlock.SetColor(BaseColorPropertyID, color);
        propertyBlock.SetColor(LegacyColorPropertyID, color);
        childRenderer.SetPropertyBlock(propertyBlock);
    }

    public void SetNormal()
    {
        ChangeColor(normalColor);
    }

    public void SetReachable()
    {
        ChangeColor(reachableColor);
    }

    public void SetSelected()
    {
        ChangeColor(selectedColor);
    }

    public void SetAttackable()
    {
        ChangeColor(attackableColor);
    }
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
    [Header("Configuraci n de la Grid")]
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
                Debug.LogWarning($"[GridManager] Coordenada duplicada detectada: {combatTile.Coordinates}. Se sobrescribir  con {combatTile.name}");
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
            Debug.LogWarning("[GridManager] GetNeighbours recibi  una celda null.");
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
            Debug.LogError("[GridManager] GetReachableCells recibi  unit null.");
            return reachableCells;
        }

        Queue<(GridCell cell, int distance)> queue = new Queue<(GridCell, int)>();
        HashSet<Vector2Int> visited = new HashSet<Vector2Int>();

        GridCell startCell = GetCell(unit.GridPosition);

        if (startCell == null)
        {
            Debug.LogError($"[GridManager] No existe startCell para unit {unit.name} en {unit.GridPosition}");
            return reachableCells;
        }

        queue.Enqueue((startCell, 0));
        visited.Add(startCell.Coordinates);

        while (queue.Count > 0)
        {
            (GridCell current, int distance) = queue.Dequeue();

            if (distance > 0)
                reachableCells.Add(current);

            if (distance >= unit.MoveRange)
                continue;

            foreach (GridCell neighbour in GetNeighbours(current))
            {
                if (neighbour == null)
                    continue;

                if (visited.Contains(neighbour.Coordinates))
                    continue;

                if (!neighbour.Walkable)
                    continue;

                if (neighbour.IsOccupied())
                    continue;

                visited.Add(neighbour.Coordinates);
                queue.Enqueue((neighbour, distance + 1));
            }
        }

        return reachableCells;
    }

    public List<GridCell> FindPath(UnitController unit, GridCell startCell, GridCell targetCell)
    {
        if (unit == null || startCell == null || targetCell == null)
            return null;

        Queue<GridCell> queue = new Queue<GridCell>();
        Dictionary<GridCell, GridCell> cameFrom = new Dictionary<GridCell, GridCell>();
        Dictionary<GridCell, int> distance = new Dictionary<GridCell, int>();

        queue.Enqueue(startCell);
        cameFrom[startCell] = null;
        distance[startCell] = 0;

        while (queue.Count > 0)
        {
            GridCell current = queue.Dequeue();

            if (current == targetCell)
                break;

            foreach (GridCell neighbour in GetNeighbours(current))
            {
                if (neighbour == null)
                    continue;

                if (!neighbour.Walkable)
                    continue;

                bool occupiedByOther = neighbour.IsOccupied() && neighbour != targetCell;
                if (occupiedByOther)
                    continue;

                if (cameFrom.ContainsKey(neighbour))
                    continue;

                int nextDistance = distance[current] + 1;
                if (nextDistance > unit.MoveRange)
                    continue;

                cameFrom[neighbour] = current;
                distance[neighbour] = nextDistance;
                queue.Enqueue(neighbour);
            }
        }

        if (!cameFrom.ContainsKey(targetCell))
            return null;

        List<GridCell> path = new List<GridCell>();
        GridCell pathCurrent = targetCell;

        while (pathCurrent != null)
        {
            path.Add(pathCurrent);
            pathCurrent = cameFrom[pathCurrent];
        }

        path.Reverse();
        return path;
    }

    public List<GridCell> GetShootableCells(UnitController attacker)
    {
        List<GridCell> shootableCells = new List<GridCell>();

        if (attacker == null)
            return shootableCells;

        GridCell attackerCell = GetCell(attacker.GridPosition);
        if (attackerCell == null)
            return shootableCells;

        foreach (GridCell cell in grid.Values)
        {
            if (cell == null)
                continue;

            if (!cell.IsOccupied())
                continue;

            if (cell.Occupant == null)
                continue;

            if (cell.Occupant.IsEnemy == attacker.IsEnemy)
                continue;

            int distance = Mathf.Abs(cell.Coordinates.x - attackerCell.Coordinates.x) / gridStep
                         + Mathf.Abs(cell.Coordinates.y - attackerCell.Coordinates.y) / gridStep;

            if (distance <= attacker.ShootRange)
            {
                shootableCells.Add(cell);
            }
        }

        return shootableCells;
    }

    public CoverType GetCoverAgainstAttacker(UnitController attacker, UnitController target)
    {
        if (attacker == null || target == null)
            return CoverType.None;

        GridCell attackerCell = GetCell(attacker.GridPosition);
        GridCell targetCell = GetCell(target.GridPosition);

        if (attackerCell == null || targetCell == null)
            return CoverType.None;

        Vector2Int delta = targetCell.Coordinates - attackerCell.Coordinates;

        if (Mathf.Abs(delta.x) > Mathf.Abs(delta.y))
        {
            if (delta.x > 0)
                return targetCell.GetCoverFromDirection(CoverDirection.Left);
            else
                return targetCell.GetCoverFromDirection(CoverDirection.Right);
        }
        else
        {
            if (delta.y > 0)
                return targetCell.GetCoverFromDirection(CoverDirection.Down);
            else
                return targetCell.GetCoverFromDirection(CoverDirection.Up);
        }
    }

    public string GetCoverDescriptionAgainstAttacker(UnitController attacker, UnitController target)
    {
        CoverType cover = GetCoverAgainstAttacker(attacker, target);

        switch (cover)
        {
            case CoverType.Full:
                return "Cobertura completa";
            case CoverType.Half:
                return "Media cobertura";
            default:
                return "Sin cobertura";
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
        if (cells == null)
            return;

        foreach (GridCell cell in cells)
        {
            if (cell != null)
                cell.SetReachableVisual();
        }
    }

    public void HighlightAttackableCells(List<GridCell> cells)
    {
        if (cells == null)
            return;

        foreach (GridCell cell in cells)
        {
            if (cell != null)
                cell.SetAttackableVisual();
        }
    }

    public bool ContainsCell(Vector2Int coords)
    {
        return grid.ContainsKey(coords);
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
using System.Collections.Generic;

public class UnitController : MonoBehaviour
{
    [SerializeField] private int moveRange = 4;
    [SerializeField] private int shootRange = 5;
    [SerializeField] private int maxHealth = 3;
    [SerializeField] private bool isEnemy = false;
    [SerializeField] private float moveStepDuration = 0.16f;

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

    public IEnumerator MoveAlongPath(List<GridCell> path, GridManager gridManager)
    {
        if (path == null || path.Count < 2)
        {
            Debug.LogWarning($"[UnitController] {name} no tiene una ruta v lida para moverse.");
            yield break;
        }

        if (gridManager == null)
        {
            Debug.LogError($"[UnitController] {name} no puede moverse: gridManager es null.");
            yield break;
        }

        GridCell startCell = gridManager.GetCell(GridPosition);
        if (startCell == null)
        {
            Debug.LogError($"[UnitController] {name} no puede moverse: startCell es null.");
            yield break;
        }

        startCell.ClearOccupant();

        for (int i = 1; i < path.Count; i++)
        {
            GridCell nextCell = path[i];

            if (nextCell == null)
                yield break;

            Vector3 start = transform.position;
            Vector3 end = gridManager.GetWorldPosition(nextCell.Coordinates) + Vector3.up * 0.5f;

            float t = 0f;

            while (t < moveStepDuration)
            {
                t += Time.deltaTime;
                float normalized = Mathf.Clamp01(t / moveStepDuration);
                transform.position = Vector3.Lerp(start, end, normalized);
                yield return null;
            }

            transform.position = end;
            GridPosition = nextCell.Coordinates;
        }

        GridCell finalCell = gridManager.GetCell(GridPosition);
        if (finalCell != null)
        {
            finalCell.SetOccupant(this);
        }

        Debug.Log($"[UnitController] {name} movido por ruta a GridPosition={GridPosition}, world={transform.position}");
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

        Debug.Log($"[UnitController] {name} recibe {damage} de da o. Vida restante: {CurrentHealth}");

        if (CurrentHealth <= 0)
        {
            Debug.Log($"[UnitController] {name} ha muerto.");

            GridCell currentCell = FindCurrentCell();
            if (currentCell != null && currentCell.Occupant == this)
            {
                currentCell.ClearOccupant();
            }

            Destroy(gameObject);
        }
    }

    private GridCell FindCurrentCell()
    {
        GridManager gridManager = FindFirstObjectByType<GridManager>();
        if (gridManager == null)
            return null;

        return gridManager.GetCell(GridPosition);
    }
}
```

