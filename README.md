# Unity-2D-Trajectory-Prediction

using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class BebeSystem : MonoBehaviour
{
    public float SpeedMultiplayer
    {
        get
        {
            if (Distance == 0)
                return 0;

            var zto = Helpers.ScaledValueToZeroAndOne(Distance, 0, 1);
            return zto;
        }
    }

    public Vector2 Direction
    {
        get
        {
            var d = Helpers.GetMouseWorldPoint() - transform.position;
            var newX = Helpers.ScaledValueToZeroAndOne(d.x, -3, 3);
            var newY = Helpers.ScaledValueToZeroAndOne(d.y, -3, 3);
            var newV = new Vector2(newX * -1, newY * -1);
            return newV;
        }
    }

    public Vector2 Velocity
    {
        get
        {
            return Vector2.ClampMagnitude(maxSpeed * SpeedMultiplayer * Direction, maxSpeed);
        }
    }

    public float Speed
    {
        get
        {
            var aa = rb2.velocity.x * rb2.velocity.x;
            var bb = rb2.velocity.y * rb2.velocity.y;
            var c = Mathf.Sqrt(aa + bb);
            return c;
        }
    }

    public float Distance
    {
        get
        {
            var d = Vector3.Distance(transform.position, Helpers.GetMouseWorldPoint());
            return Mathf.Clamp(d, 0, radius);
        }
    }

    public float WindVolume
    {
        get
        {
            if(Speed > maxSpeed / 2)
            {
                var volume = Helpers.ScaledValueToZeroAndOne(Speed, 0, maxSpeed);
                return volume;
            }
            return 0;
        }
    }

    public float maxSpeed;
    public float radius;
    public float maxAngularVelocity;
    public float maxY;
    [HideInInspector] public int touchCount;
    public bool canThrow;
    public bool clickDown;
    public bool clickDrag;
    

    private float g;
    private float volume;

    public int curveResolution;
    public int rayResolution;
    public LayerMask hitLayer;

    private Rigidbody2D rb2;
    private LineRenderer curve;
    private DeathManager deathManager;
    private PauseManager pauseManager;
    private ClickBebeManager clickBebe;
    private SlowMotion slowMotion;

    private void Awake()
    {
        rb2 = GetComponent<Rigidbody2D>();
        curve = GameObject.FindObjectOfType<LineRenderer>();
        deathManager = GameObject.FindObjectOfType<DeathManager>();
        pauseManager = GameObject.FindObjectOfType<PauseManager>();
        clickBebe = GameObject.FindObjectOfType<ClickBebeManager>();
        slowMotion = GameObject.FindObjectOfType<SlowMotion>();
    }

    private void Start()
    {
        Physics2D.queriesHitTriggers = true;
        g = Mathf.Abs(Physics2D.gravity.y);
        AudioManager.instance.Play("Whoosh");
        AudioManager.instance.SetVolume("Whoosh", 0);
        AudioManager.instance.PlayTransition("Baby Drop SFX", 1f);
    }

    private void Update()
    {
        volume = Mathf.MoveTowards(volume, WindVolume, Time.deltaTime * 0.5f);
        AudioManager.instance.SetVolume("Whoosh", volume);

        if (Input.GetMouseButtonDown(0) && clickDown)
        {
            if (!deathManager.gameOver && !pauseManager.pause)
            {
                curve.enabled = true;
                curve.positionCount = 0;
                rb2.bodyType = RigidbodyType2D.Static;
                touchCount++;
            }
        }
        
        if (Input.GetMouseButton(0) && clickDown || clickDrag)
        {
            clickDrag = true;
            clickDown = false;

            if (!deathManager.gameOver && !pauseManager.pause)
            {
                slowMotion.slowMotion = true;
            }
            
            if (!deathManager.gameOver && !pauseManager.pause)
            {
                StartCoroutine(RenderArc());
            }
        }
        
        if(Input.GetMouseButtonUp(0) && clickDrag)
        {
            if (!deathManager.gameOver && !pauseManager.pause)
            {
                rb2.bodyType = RigidbodyType2D.Dynamic;
                rb2.velocity = Velocity;
                var r = Random.Range(-1.0f, 1.0f);
                rb2.angularVelocity = maxAngularVelocity * r;
                StartCoroutine(RenderArc());
                AudioLowManager.instance.enableAudio = false;
                Time.timeScale = 1;
                AudioManager.instance.Play("Bebe Shoot SFX");
            }

            slowMotion.slowMotion = false;
            slowMotion.EndSlowMo();
            curve.enabled = false;
            clickDrag = false;
            clickBebe.clickUp = true;
        }
    }

    private float MaxTimeX()
    {
        var x = Velocity.x;
        var t = (HitPosition().x - transform.position.x) / x;
        return t;
    }

    private float MaxTimeY()
    {
        var v = Velocity.y;
        var vv = v * v;

        var t = (v + Mathf.Sqrt(vv + 2 * g * (transform.position.y - maxY))) / g;
        return t;
    }

    private IEnumerator RenderArc()
    {
        curve.positionCount = curveResolution + 1;
        curve.SetPositions(CalculateArcArray());
        yield return null;
    }

    private Vector3[] CalculateArcArray()
    {
        Vector3[] arcArray = new Vector3[curveResolution + 1];

        var add = MaxTimeX() / curveResolution;


        for (int i = 0; i <= curveResolution; i++)
        {   
            var t = add * i;
            arcArray[i] = CalculateArcPoint(t);
        }

        return arcArray;
    }

    private Vector2 HitPosition()
    {
        var add = MaxTimeY() / rayResolution;

        for (int i = 0; i <= rayResolution; i++)
        {
            var t = add * i;
            var tt = add * (i + 1);

            var hit = Physics2D.Linecast(CalculateArcPoint(t), CalculateArcPoint(tt), hitLayer);

            if(hit)
            {
                return hit.point;
            }
        }
        return CalculateArcPoint(MaxTimeY());
    }

    private Vector3 CalculateArcPoint(float t)
    {
        float x = Velocity.x * t;
        float y = (Velocity.y * t) - (g * Mathf.Pow(t, 2) / 2);
        return new Vector3(x + transform.position.x, y + transform.position.y);
    }

    private void OnDrawGizmos()
    {
        Gizmos.color = Color.red;
        Gizmos.DrawWireSphere(transform.position, radius);
    }
}
