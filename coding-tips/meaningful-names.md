# Meaningful Names.

Naming your classes and methods in a meaning way helps:
- Break down code by responsibilities.
- Read and understand code faster.
- Keep code blocks cohesive and focused.

## The story of "`NPCController`" ... a class that did too much.

- The `NPCController` script checks the NPC's current state and sets a behavior flag the behavior tree can use. Then checks to see if the player is detected, and depending on the current state or if the player is detected, then it sets the NPC speed (for locomotion and animation).


```csharp
public class NPCController : MonoBehaviour
{
    public float CurrentSpeed { get { return aiPath.velocity.magnitude; } }

    [SerializeField] private Animator animator;
    [SerializeField] private AIPath aiPath;
    [SerializeField] private BehaviorTree behaviorTree;
    [SerializeField] private StuckHandler stuckHandler;
    [SerializeField] private Rigidbody rigidBody;

    private bool hungry = false;
    private bool angry = false;
    private bool playerDetected = false;

    // Behavior Tree Variables
    private string BTVAR_NPC_SPEED_WALK = "NPCSpeed_Walk"; // BT variable name
    private string BTVAR_NPC_SPEED_RUN = "NPCSpeed_Run";
    private string BTVAR_NPC_STATE = "NPCState";
    private string BTVAR_PLAYER_DETECTED = "PlayerDetected";

    private void Update()
    {
        CheckNPCState();
        SetBehaviorTreeNPCState();
        CheckPlayerDetected();

        if (CurrentState == NPCState.Calm && playerDetected)
        {
            aiPath.canMove = false;
            SetNPCSpeed(idleSpeed);
            animator.SetFloat(ANIMVAR_SPEED, idleSpeed);
            Vector3 direction = (behaviorTree.GetVariable("player") as SharedGameObject).Value.transform.position - transform.position;
            direction.y = 0;
            transform.rotation = Quaternion.Slerp(transform.rotation, Quaternion.LookRotation(direction), 0.1f);
        }
        else
        {
            aiPath.canMove = true;
            if (CurrentState == NPCState.Hangry)
            {
                SetNPCSpeed(runSpeed);
            }
            else
            {
                SetNPCSpeed(walkSpeed);
            }
            animator.SetFloat(ANIMVAR_SPEED, CurrentSpeed);
        }
    }

    public void SetHungry(bool value)
    {
        hungry = value;
    }

    public void SetAngry(bool value)
    {
        angry = value;
    }

    private void CheckNPCState()
    {
        if (hungry && angry)
        {
            SetNPCState(NPCState.Hangry);
        }
        else if (hungry)
        {
            SetNPCState(NPCState.Hungry);
        }
        else if (angry)
        {
            SetNPCState(NPCState.Angry);
        }
        else
        {
            SetNPCState(NPCState.Calm);
        }
    }

    private void CheckPlayerDetected()
    {
        var btVarPlayerDetected = (SharedBool)behaviorTree.GetVariable(BTVAR_PLAYER_DETECTED);
        playerDetected = btVarPlayerDetected.Value;
    }

    private void SetNPCState(NPCState newState)
    {
        CurrentState = newState;

        if (newState == NPCState.Hungry)
        {
            hungry = true;
            angry = false;
        }
        else if (newState == NPCState.Angry)
        {
            hungry = false;
            angry = true;
        }
        else if (newState == NPCState.Hangry)
        {
            hungry = true;
            angry = true;
        }
        else
        {
            hungry = false;
            angry = false;
        }
    }

    private void SetBehaviorTreeVariable(string variableName, object value)
    {
        behaviorTree.SetVariableValue(variableName, value);
    }

    private void SetBehaviorTreeNPCState()
    {
        SetBehaviorTreeVariable(BTVAR_NPC_STATE, CurrentState);
    }

    private void CreatePatrolWaypoints()
    {
        ClearWaypoints();
        patrolWaypoints.Clear();
        float minWaypointDistance = 6f;
        float maxWaypointDistance = 10f;
        CurrentState = NPCState.Calm;
        // generate 3 - 5 random waypoints around the NPC with a max distance of 5 units
        int totalWaypoints = Random.Range(3, 6);
        for (int i = 0; i < totalWaypoints; i++)
        {
            Vector3 randomDirection = Random.insideUnitSphere;
            randomDirection.y = 0;
            randomDirection.Normalize();
            float waypointDistance = Random.Range(minWaypointDistance, maxWaypointDistance);
            randomDirection *= waypointDistance;
            randomDirection += transform.position;
            GameObject waypoint = new GameObject($"{gameObject.name} Waypoint {i}");
            waypoint.transform.position = new Vector3(randomDirection.x, transform.position.y, randomDirection.z);
            patrolWaypoints.Add(waypoint);
        }

        StartPatrol(patrolWaypoints);
    }

    private void StartPatrol(List<GameObject> waypoints)
    {
        aiPath.slowdownDistance = patrolStopDistance;
        aiPath.endReachedDistance = patrolStopDistance;
        SetNPCSpeed(walkSpeed);
        animator.SetTrigger(ANIMTRIG_MOVE);
        behaviorTree.SetVariableValue(BTVAR_PATROL_WAYPOINTS, waypoints);
        behaviorTree.SendEvent<object>(BTEVENT_START_PATROL, waypoints);
    }

    private void ClearWaypoints()
    {
        foreach (GameObject waypoint in patrolWaypoints)
        {
            Destroy(waypoint);
        }
        patrolWaypoints.Clear();
    }



    private void SetNPCSpeed(float npcSpeed)
    {
        aiPath.maxSpeed = npcSpeed;
        behaviorTree.SetVariableValue(BTVAR_SPEED, npcSpeed);
    }

    public void KnockBack(Vector3 direction, float force)
    {
        animator.SetTrigger(ANIMTRIG_GET_HIT);

        if (rigidBody != null)
        {
            float appliedForce = force * rigidBody.mass;
            rigidBody.AddForce(direction * appliedForce, ForceMode.Impulse);
        }
    }

    public void OnCollisionEnter(Collision collision)
    {
        if (collision.gameObject.CompareTag("Player"))
        {
            Vector3 direction = collision.transform.position - transform.position;
            direction.y = 3;
            direction.Normalize();
            KnockBack(direction, 10f);
        }
    }

}

```

- Okay to start, but can be improved.
- One easy way to spot that the script is doing TOO MUCH: __read its description__.

> The `NPCController` script checks the NPC's current state **AND** sets a behavior flag the behavior tree can use. **THEN** checks to see if the player is detected, **AND** depending on the current state **OR** if the player is detected, **THEN** it sets the NPC speed (for locomotion **AND** animation).

- It has one good thing going for it: **MEANINGFUL NAMES**.

Meaningful names like:

- `SetHungry`
- `SetAngry`
- `CheckPlayerDetected`
- `SetNPCState`
- `CreatePatrolWaypoints`
- `StartPatrol`
- `SetNPCSpeed`

helped me:

1) Identify that there are too many unrelated RESPONSIBILITIES in this class.
2) Break down the script into smaller cohesive classes.
3) Keep functions focused on as few actions as possible.

# Refactored Scripts

## Refactored Script Components
- `AngerSetter`
- `HungerSetter`
- `TargetSetter`
- `PatrolDestinationSetter`
- `LocomotionSpeedSetter`
- ... others

### AngerSetter
**Description**: Sets the "Angry" property based on a countdown timer or if the target is a "Hero" NPC.

```csharp
public class AngerSetter : MonoBehaviour
{
    [Header("References")]
    [SerializeField, RequireDependency(typeof(HungerSetter))] private HungerSetter hungerSetter;
    [SerializeField, RequireDependency(typeof(TargetSetter))] private TargetSetter targetSetter;

    public bool IsAngry { get { return angry; } } // needed for behavior tree

    private bool angry = false;
    private float secondsUntilAngry;
    private float countDownToAngry;
    private System.Random randomizer = new System.Random();

    private void OnEnable()
    {
        bool isAngryAtStart = GetIsAngryByRandomChance();
        angry = isAngryAtStart;
        secondsUntilAngry = GetRandomSecondsUntilAngry();
        countDownToAngry = angry ? 0f : secondsUntilAngry;
    }

    private bool GetIsAngryByRandomChance()
    {
        int percentChanceOfAngryAtStart = 5;
        int randomInt = randomizer.Next(0, 100);
        return randomInt < percentChanceOfAngryAtStart;
    }

    private float GetRandomSecondsUntilAngry()
    {
        int minSecondsUntilAngry = 7;
        int maxSecondsUntilAngry = 30;
        return (float)randomizer.Next(minSecondsUntilAngry, maxSecondsUntilAngry);
    }

    private void Update()
    {
        bool isTargetingHero = targetSetter.TargetType == NPCTargetType.Hero.ToString();
        if (isTargetingHero)
        {
            SetAngry();
            return;
        }

        bool shouldCountDownToAngry = ShouldCountDownToAngry();
        if (shouldCountDownToAngry)
        {
            CountDownUntilAngry();
        }
    }
    
    private void SetAngry()
    {
        angry = true;
        inAngerCoolDownPeriod = false;
        countDownToAngry = 0f;
        angerEvent?.Invoke();
    }

    private bool ShouldCountDownToAngry()
    {
        if (inAngerCoolDownPeriod)
        {
            return false;
        }
        else if (hungerSetter.IsHungry)
        {
            return true;
        }
        return false;
    }

    private void CountDownUntilAngry()
    {
        if (countDownToAngry > 0f)
        {
            countDownToAngry -= Time.deltaTime;
            if (countDownToAngry <= 0f)
            {
                SetAngry();
            }
        }
    }

    public float GetAngerPercentage()
    {
        if (angry)
        {
            return 1f;
        }
        else if (!hungerSetter.IsHungry)
        {
            return 0f;
        }

        float angerPercentage = 1 - countDownToAngry / secondsUntilAngry;

        return Mathf.Clamp(angerPercentage, 0f, 1f);
    }

    private void ResetAnger()
    {
        angry = false;
        inAngerCoolDownPeriod = true;
        StartCoroutine(StopAngerCoolDownPeriod());
    }
}
```


### LocomotionSpeedSetter
**Description**: Sets the locomotion speed.
```csharp
public class LocomotionSpeedSetter : MonoBehaviour
{
    [Header("References")]
    [SerializeField, RequireDependency(typeof(AIPath))] private AIPath aiPath;
    [SerializeField, RequireDependency(typeof(AnimatorController))] private AnimatorController animatorController;
    [SerializeField, RequireDependency(typeof(StateSetter))] private StateSetter stateSetter;

    [Header("Script Variables")]
    [SerializeField, Range(1, 5)] private float normalSpeed = 2.5f;
    [SerializeField, Range(4, 10)] private float fastSpeed = 8f;

    public float CurrentSpeed { get => GetCurrentSpeed(); } // needed for behavior tree
    public bool IsLocomotionEnabled { get => isLocomotionEnabled; }

    private float currentSpeed;
    private bool isLocomotionEnabled = true;

    private void OnEnable()
    {
        currentSpeed = 0f;
        EnableLocomotion();
    }

    private void Update()
    {
        if (isLocomotionEnabled)
        {
          SetAiPathMaxSpeedByState();
          SetCurrentSpeed();
        }
    }

    private void SetAiPathMaxSpeedByState()
    {
        if (stateSetter.IsAngry())
        {
            aiPath.maxSpeed = fastSpeed;
        }
        else
        {
            aiPath.maxSpeed = normalSpeed;
        }
    }

    private void SetCurrentSpeed()
    {
        float thresholdMovementSpeed = 0.2f;
        if (aiPath.velocity.magnitude < thresholdMovementSpeed)
        {
            currentSpeed = 0f;
        }
        else
        {
            currentSpeed = aiPath.velocity.magnitude;
        }
    }

    public float GetCurrentSpeed()
    {
        return currentSpeed;
    }

    public void EnableLocomotion()
    {
        isLocomotionEnabled = true;
        aiPath.canMove = true;
    }

    public void DisableLocomotion()
    {
        aiPath.maxSpeed = 0f;
        aiPath.canMove = false;
        isLocomotionEnabled = false;
    }
}
```


### PatrolDestinationSetter
**Description**: Creates and sets the next patrol destination for NPC.
```csharp
public class PatrolDestinationSetter : MonoBehaviour
{
    [Header("References")]
    [SerializeField, RequireDependency(typeof(AIPath))] private AIPath aiPath;
    [SerializeField, RequireDependency(typeof(TargetSetter))] private TargetSetter targetSetter;

    private float secondsToWaitWhenReachedEndOfPath = 3f;
    private float timeSinceReachedEndOfPath = 0f;

    private void OnEnable()
    {
        aiPath.canMove = true;
    }

    private void Update()
    {
        CountUpFromEndOfPath();
        bool shouldSetNewDestination = ShouldSetNewDestination();
        if (shouldSetNewDestination)
        {
            Vector3 destination = GetNewDestination();
            SetNextDestination(destination);
        }
    }

    private void CountUpFromEndOfPath()
    {
        if (aiPath.reachedEndOfPath)
        {
            timeSinceReachedEndOfPath += Time.deltaTime;
        }
    }

    private bool ShouldSetNewDestination()
    {
        bool hasTarget = targetSetter.Target != null;
        bool hasAiPathDestination = HasAiPathDestination();
        bool hasReachedEndOfPath = aiPath.reachedEndOfPath;
        bool destinationIsUnset = !hasTarget && !hasAiPathDestination;

        if (hasReachedEndOfPath && timeSinceReachedEndOfPath < secondsToWaitWhenReachedEndOfPath)
        {
            return false;
        }
        else if (!hasTarget && hasReachedEndOfPath)
        {
            return true;
        }
        else if (destinationIsUnset)
        {
            return true;
        }
        return false;
    }

    private bool HasAiPathDestination()
    {
        if (aiPath == null)
        {
            return false;
        }

        bool infiniteX = aiPath.destination.x == Mathf.Infinity;
        bool infiniteY = aiPath.destination.y == Mathf.Infinity;
        bool infiniteZ = aiPath.destination.z == Mathf.Infinity;
        return !infiniteX && !infiniteY && !infiniteZ;
    }

    private Vector3 GetNewDestination()
    {
        timeSinceReachedEndOfPath = 0f;
        if (targetSetter.Target)
        {
            return targetSetter.Target.transform.position;
        }
        return GetRandomPointOnSamePlane();

    }

    private Vector3 GetRandomPointOnSamePlane()
    {
        Vector3 randomPoint = Random.insideUnitSphere * 10;
        int attempts = 0;
        while (randomPoint.magnitude < 4 && attempts < 5)
        {
            randomPoint = Random.insideUnitSphere * 10;
            attempts++;
        }

        randomPoint += transform.position;
        randomPoint.y = transform.position.y;
        return randomPoint;
    }

    public void SetNextDestination(Vector3 destination)
    {
        aiPath.destination = destination; // 1 line functions - judgement call
    }
}
```


# Naming Tips

- Be consistent with your naming.
- Be explicit about your names.
- Use nouns for classes, verbs for methods.
- Leave wiggle room if needed.

### Be consistent.

Instead of having methods like:
- `SetPower`
- `StoreLastKnownPlayerPosition`
- `SaveHomePosition`

try:

- `SetPower`, `SetLastKnownPlayerPosition`, `SetHomePosition`, or
- `StorePower`, `StoreLastKnownPlayerPosition`, `StoreHomePosition`, or
- `SavePower`, `SaveLastKnownPlayerPosition`, `SaveHomePosition`

Instead of having classes like:

-  `PowerUpController`
-  `HitHandler`
-  `NPCManager`

try:
- `PowerUpController`, `HitController`, `NPCController`, or
- `PowerUpHandler`, `HitHandler`, `NPCHandler`, or
- `PowerUpManager`, `HitManager`, `NPCManager`

(UNLESS "CONTROLLER", "HANDLER" or "MANAGER" actually mean different things in your project.)

As a reader you will develop expectations of what methods with a certain prefix DO or do NOT DO.

This is why even if the `SetNextDestination` method is a one-liner that can be incorporated into another method, I wanted to keep the expectation for `GetNewDestination` and `SetNextDestination` separate (my `Get....` methods should not SET anything)

### Be explicit about your names.

Instead of having variables like:

- `threshold`
- `value`
- `delay`

try:

- `healthThreshold`
- `itemPrice`
- `secondsBeforeRespawn`

This helps clarify the context without having to search for the variable declaration.

### Use nouns for classes, verbs for methods.

Classes in my project are named after the responsibility they have.
Methods are named after the focused action they perform.
I think that's easy to understand when reading the code:

```csharp
HungerSetter
  - GetRandomSecondsUntilHungry()
  - CountDownUntilHungry()
  - CountUpSecondsSinceHungry()
  - SetHungry()
  - GetHungerPercentage()
  - ResetHunger()

HealthController
  - ResetHealth()
  - GetHealthPercentage()
  - DecreaseHealth()
  - IsAlive()
```

### Leave wiggle room if needed (for game development).

Some times, it's okay to break these guidelines, but try to make that the exception.

This function does 2 things, but it was more efficient to keep it together than to create a new event for the second action that will not be used anywhere else.

```csharp
private void ApplyHitDamageAndNotifyIfKilled(float damage, AttackController attacker)
{
    bool wasTargetAlive = healthController.IsAlive();
    healthController.DecreaseHealthByAmount(damage);
    bool isTargetDead = !healthController.IsAlive();
    if (wasTargetAlive && isTargetDead)
    {
        attacker.KilledTarget();
    }
}
```

Note that the name is still descriptive of what the multiple actions done so that there's no confusion with the expectations of the method if it was just named `ApplyHitDamage`.

Don't let these opinionated guidelines hold you back from writing code that works for your project.