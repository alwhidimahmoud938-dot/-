public static class Constants
{
    public static readonly string PlayerTag = "Player";
    public static readonly string AnimationStarted = "started";
    public static readonly string AnimationJump = "jump";
    public static readonly string WidePathBorderTag = "WidePathBorder";
    public static readonly string StatusTapToStart = "Tap to start";
    public static readonly string StatusDeadTapToStart = "Dead. Tap to start";
}
```<grok:render card_id="81274f" card_type="citation_card" type="render_inline_citation"><argument name="citation_id">74</argument></grok:render>

#### **2. GameState.cs** (حالة اللعبة - أنشئ enum)
```csharp
public enum GameState { Start, Playing, Dead }
```<grok:render card_id="8808d9" card_type="citation_card" type="render_inline_citation"><argument name="citation_id">63</argument></grok:render>

#### **3. GameManager.cs** (Singleton لإدارة اللعبة)
```csharp
using UnityEngine;

public class GameManager : MonoBehaviour
{
    private static GameManager instance;
    public static GameManager Instance => instance ??= new GameManager();

    void Awake()
    {
        if (instance == null) instance = this;
        else DestroyImmediate(this);
    }

    protected GameManager()
    {
        GameState = GameState.Start;
        CanSwipe = false;
    }

    public GameState GameState { get; set; }
    public bool CanSwipe { get; set; }

    public void Die()
    {
        UIManager.Instance.SetStatus(Constants.StatusDeadTapToStart);
        GameState = GameState.Dead;
    }
}
```<grok:render card_id="857fac" card_type="citation_card" type="render_inline_citation"><argument name="citation_id">73</argument></grok:render>

#### **4. UIManager.cs** (واجهة المستخدم - أضف إلى Canvas)
```csharp
using UnityEngine;
using UnityEngine.UI;

public class UIManager : MonoBehaviour
{
    private static UIManager instance;
    public static UIManager Instance => instance ??= new UIManager();

    void Awake()
    {
        if (instance == null) instance = this;
        else DestroyImmediate(this);
    }

    private float score = 0;
    public Text ScoreText, StatusText;

    public void ResetScore() { score = 0; UpdateScoreText(); }
    public void IncreaseScore(float value) { score += value; UpdateScoreText(); }
    public void SetStatus(string text) { StatusText.text = text; }

    private void UpdateScoreText() { ScoreText.text = score.ToString(); }
}
```<grok:render card_id="ea6f60" card_type="citation_card" type="render_inline_citation"><argument name="citation_id">71</argument></grok:render>

#### **5. PlayerMovement.cs** (حركة الشخصية - أضف إلى Character، استخدم Keyboard للبساطة)
```csharp
using UnityEngine;
using UnityEngine.SceneManagement;

public class PlayerMovement : MonoBehaviour // استبدل CharacterSidewaysMovement
{
    private Vector3 moveDirection = Vector3.zero;
    public float gravity = 20f, jumpSpeed = 8f, speed = 6f, sideSpeed = 5f;
    private CharacterController controller;
    private Animator anim;

    private bool isChangingLane = false;
    private Vector3 targetLane;
    private Vector3 sidewaysDistance = Vector3.right * 2f; // مسافة المسار

    void Start()
    {
        controller = GetComponent<CharacterController>();
        anim = GetComponentInChildren<Animator>();
        moveDirection = transform.forward * speed;
        UIManager.Instance.ResetScore();
        UIManager.Instance.SetStatus(Constants.StatusTapToStart);
        GameManager.Instance.GameState = GameState.Start;
    }

    void Update()
    {
        switch (GameManager.Instance.GameState)
        {
            case GameState.Start:
                if (Input.GetMouseButtonUp(0) || Input.GetKeyDown(KeyCode.Space))
                {
                    anim.SetBool(Constants.AnimationStarted, true);
                    GameManager.Instance.GameState = GameState.Playing;
                    UIManager.Instance.SetStatus("");
                }
                break;

            case GameState.Playing:
                UIManager.Instance.IncreaseScore(0.01f); // نقاط مستمرة
                HandleMovement();
                break;

            case GameState.Dead:
                anim.SetBool(Constants.AnimationStarted, false);
                if (Input.GetMouseButtonUp(0) || Input.GetKeyDown(KeyCode.Space))
                    SceneManager.LoadScene(SceneManager.GetActiveScene().name);
                break;
        }

        moveDirection.y -= gravity * Time.deltaTime;
        controller.Move(moveDirection * Time.deltaTime);
    }

    void HandleMovement()
    {
        if (controller.isGrounded && Input.GetKeyDown(KeyCode.Space))
        {
            moveDirection.y = jumpSpeed;
            anim.SetBool(Constants.AnimationJump, true);
        } else {
            anim.SetBool(Constants.AnimationJump, false);
        }

        // تغيير مسار: A/D أو Left/Right
        if (controller.isGrounded && !isChangingLane)
        {
            if (Input.GetKeyDown(KeyCode.A) || Input.GetKeyDown(KeyCode.LeftArrow))
            {
                isChangingLane = true;
                targetLane = transform.position - sidewaysDistance;
                moveDirection.x = -sideSpeed;
            }
            else if (Input.GetKeyDown(KeyCode.D) || Input.GetKeyDown(KeyCode.RightArrow))
            {
                isChangingLane = true;
                targetLane = transform.position + sidewaysDistance;
                moveDirection.x = sideSpeed;
            }
        }

        if (isChangingLane && Mathf.Abs(transform.position.x - targetLane.x) < 0.1f)
        {
            isChangingLane = false;
            moveDirection.x = 0;
        }
    }

    public void OnControllerColliderHit(ControllerColliderHit hit)
    {
        if (hit.gameObject.CompareTag(Constants.WidePathBorderTag))
        {
            isChangingLane = false;
            moveDirection.x = 0;
        }
    }
}
```<grok:render card_id="5c18e9" card_type="citation_card" type="render_inline_citation"><argument name="citation_id">67</argument></grok:render><grok:render card_id="99c8fd" card_type="citation_card" type="render_inline_citation"><argument name="citation_id">68</argument></grok:render>

#### **6. PathSpawnCollider.cs** (مولد الطرق الجديدة - أضف إلى PathSpawnCollider)
```csharp
using UnityEngine;

public class PathSpawnCollider : MonoBehaviour
{
    public float positionY = 0.81f;
    public Transform[] PathSpawnPoints; // 3 نقاط: straight, left, right
    public GameObject Path; // Prefab الطريق
    public GameObject DangerousBorder; // Prefab الحدود المميتة

    void OnTriggerEnter(Collider hit)
    {
        if (hit.CompareTag(Constants.PlayerTag))
        {
            int randomPoint = Random.Range(0, PathSpawnPoints.Length);
            for (int i = 0; i < PathSpawnPoints.Length; i++)
            {
                if (i == randomPoint)
                    Instantiate(Path, PathSpawnPoints[i].position, PathSpawnPoints[i].rotation);
                else
                {
                    Vector3 rot = PathSpawnPoints[i].rotation.eulerAngles;
                    rot.y += 90;
                    Vector3 pos = PathSpawnPoints[i].position;
                    pos.y += positionY;
                    Instantiate(DangerousBorder, pos, Quaternion.Euler(rot));
                }
            }
        }
    }
}
```<grok:render card_id="6d5bc6" card_type="citation_card" type="render_inline_citation"><argument name="citation_id">69</argument></grok:render>

#### **7. StuffSpawner.cs** (مولد العملات/عقبات - أضف إلى كل Path)
```csharp
using UnityEngine;

public class StuffSpawner : MonoBehaviour
{
    public Transform[] StuffSpawnPoints; // نقاط الإنشاء
    public GameObject[] Bonus; // Prefabs عملات
    public GameObject[] Obstacles; // Prefabs عقبات
    public bool RandomX = true;
    public float minX = -2f, maxX = 2f;

    void Start()
    {
        bool placeObs = Random.Range(0, 2) == 0;
        int obsIndex = -1;
        if (placeObs)
        {
            obsIndex = Random.Range(1, StuffSpawnPoints.Length);
            CreateObject(StuffSpawnPoints[obsIndex].position, Obstacles[Random.Range(0, Obstacles.Length)]);
        }

        for (int i = 0; i < StuffSpawnPoints.Length; i++)
        {
            if (i == obsIndex) continue;
            if (Random.Range(0, 3) == 0)
                CreateObject(StuffSpawnPoints[i].position, Bonus[Random.Range(0, Bonus.Length)]);
        }
    }

    void CreateObject(Vector3 pos, GameObject prefab)
    {
        if (RandomX) pos.x += Random.Range(minX, maxX);
        Instantiate(prefab, pos, Quaternion.identity);
    }
}
```<grok:render card_id="62a09a" card_type="citation_card" type="render_inline_citation"><argument name="citation_id">70</argument></grok:render>

#### **8. Candy.cs** (عملة - أضف إلى Candy prefab، SphereCollider IsTrigger)
```csharp
using UnityEngine;

public class Candy : MonoBehaviour
{
    public int ScorePoints = 100;
    public float rotateSpeed = 50f;

    void Update() { transform.Rotate(Vector3.up * rotateSpeed * Time.deltaTime); }

    void OnTriggerEnter(Collider col)
    {
        if (col.CompareTag(Constants.PlayerTag))
        {
            UIManager.Instance.IncreaseScore(ScorePoints);
            Destroy(gameObject);
        }
    }
}
```<grok:render card_id="1a08b3" card_type="citation_card" type="render_inline_citation"><argument name="citation_id">72</argument></grok:render>

#### **9. Obstacle.cs** (عقبة - أضف إلى Obstacle prefab، BoxCollider IsTrigger)
```csharp
using UnityEngine;

public class Obstacle : MonoBehaviour
{
    void OnTriggerEnter(Collider col)
    {
        if (col.CompareTag(Constants.PlayerTag))
            GameManager.Instance.Die();
    }
}
```<grok:render card_id="894eea" card_type="citation_card" type="render_inline_citation"><argument name="citation_id">75</argument></grok:render>

#### **10. TimeDestroyer.cs** (حذف الأجسام القديمة - أضف إلى Path, Candy, Obstacle)
```csharp
using UnityEngine;

public class TimeDestroyer : MonoBehaviour
{
    public float LifeTime = 10f;

    void Start() { Invoke(nameof(DestroyObject), LifeTime); }

    void DestroyObject()
    {
        if (GameManager.Instance.GameState != GameState.Dead)
            Destroy(gameObject);
    }
}
```<grok:render card_id="73e29c" card_type="citation_card" type="render_inline_citation"><argument name="citation_id">76</argument></grok:render>

### **الخطوة 3: إعداد إضافي**
- **Animator Controller**: أنشئ states: Idle -> Run (bool "started"), Run <-> Jump (bool "jump").
- **Tags**: أضف "Player" لـ Character، "WidePathBorder" لـ DangerousBorder.
- **كاميرا تتبع**: أضف سكريبت بسيط:
```csharp
using UnityEngine;

public class CameraFollow : MonoBehaviour
{
    public Transform target;
    public Vector3 offset = new Vector3(0, 2, -5);

    void LateUpdate() { transform.position = target.position + offset; }
}
