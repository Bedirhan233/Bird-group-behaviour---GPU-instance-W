# Bird-group-behaviour---GPU-instance-W

During my internship att Gibbet games I got a task to comelete for their game Club House on Haunted Hill. THe task was to optimize their bird group system. I even added new features to group behaviour. Where the birds could land on a spot, idle on gorund, get scared, dive, a system where birds could go to fdifferent areas based on prio number of thsoe areas.

After analyzing the existing code I decided to create my own system. The big issue with their system was that every bird had a 2d spehre box that calcualted all the time. 
They also had a spehere around the group so every time the bird hit outside of the box it calcualted a new position.

I changed the system. Instead of puting the logic inside of the bird, I made a main class for all bird movement and one for the managing. 

For exameple the pulsating system: 

Then made a chain reaction with coroutine. First I generate a random point with a radius that starts from the center of that class and then sended the bird to that point with coroutine. When it reaches it asks for a new position through event system. 



![pulsating](https://github.com/user-attachments/assets/67fa8605-49a0-4743-86b3-a348535a5af5)


<img width="273" height="190" alt="image" src="https://github.com/user-attachments/assets/6f932323-d32a-44d1-966e-e90c25ceae0d" />



In my manage class I have this function that triggers whenever the MoveToTarget object moves.

```c#
Vector3 directionToParent = (oldPosOfCentralPosition - parentOfCenterPower.transform.position).normalized;

Vector3 centerPoint = parentOfCenterPower.transform.position + directionToParent * offsetValueMiddlePoint;
Debug.DrawRay(oldPosOfCentralPosition, directionToParent * 100);

birdMovement.middlePointMovement = centerPoint;
```
This creates a middle point between the target position and current position

Then I create a random spehere around that position in my movement class
```c#
Vector3 randomLocation = middlePointMovement + (Random.insideUnitSphere * 2);
bird.randomPoint = randomLocation;
```
Then in bird I have a simple movement

```c#
    public IEnumerator FlyToRandom(bool isAccelerating = false, float currentSpeed = 0)
    {

        float targetSpeed = flyingInAirSpeed;   // Final flying speed
        
        while (true)
        {
            if (!isTurning && !isDiving)
            {
                if ((transform.position - randomPoint).sqrMagnitude < maxDistanceBeforeNewPos)
                {
                    OnAskingNewRandomPoints.Invoke(this);
                }

                direction = (randomPoint - gameObject.transform.position);

                if (flappingAnimationCoroutine == null)
                {
                    flappingAnimationCoroutine = StartCoroutine(FlyingInAirAnimationPattern());
                }
            }

            if (isDiving)
            {
                //animationPlayComponent.PlayAnimation("Soar");
                if (autoAnimate)
                {
                    playAnimationComponent.PlayAnimation(AnimationType.soar, true);
                }

                isDivingTimer += Time.deltaTime;
                if (isDivingTimer > 3)
                {
                    ResettingDive();
                }
            }

            Quaternion targetRotation = Quaternion.LookRotation(direction);

            // removing this if you want to boost fps
            transform.rotation = Quaternion.Slerp(transform.rotation, targetRotation, rotationSpeed * Time.deltaTime);
            //transform.rotation = targetRotation;
            if (isAccelerating)
            {
                  currentSpeed = Mathf.Lerp(currentSpeed, targetSpeed, 0);
            }
            else
            {
                currentSpeed = flyingInAirSpeed * Random.Range(0.8f, 1.5f);  // Normal speed
            }

            transform.position += transform.forward * Time.deltaTime * currentSpeed;
            yield return null;
        }

    }

```
The birds are even reacting to obstacles that are in their way

![ForPResent2-ezgif com-resize](https://github.com/user-attachments/assets/ce61749b-6c7a-4ccd-bd78-23b6eaf07596)

Each bird sends out a raycast. If they hit the building then this logic runs
```c#
IEnumerator FlyToSide()
{
    isTurning = true;
    direction = gameObject.transform.right;

    while (Vector3.Angle(transform.forward, direction) > maxAllowedRotationDif)
    {
        Quaternion targetRotation = Quaternion.LookRotation(direction);
        transform.rotation = Quaternion.Slerp(transform.rotation, targetRotation, rotationSpeed * Time.deltaTime);
        yield return null;
    }

    // delay for resetting side logic
    yield return new WaitForSeconds(0.5f);
    isTurning = false;
    flyToSideCoroutine = null;
    ResettingDirection();
}
```
# Bird landing 

![ForPResent2](https://github.com/user-attachments/assets/135b21a7-cff3-4be5-a49a-1756cc8fabd0)


To make the birds land on a surface I make landing platforms first. 

```c#
public void GetPosOfSurface()
{
    hitPositions.Clear();

    const float minPointDistance = 0.25f;
    float minSqr = minPointDistance * minPointDistance;

    for (int j = 0; j < instensity; j++)
    {
        Vector2 randomPoint = UnityEngine.Random.insideUnitCircle * range;
        Vector3 raycastStart = new Vector3(
            randomPoint.x + transform.position.x,
            transform.position.y,
            randomPoint.y + transform.position.z
        );

        Debug.DrawRay(raycastStart, Vector3.down * rayDistance);

        if (!Physics.Raycast(raycastStart, Vector3.down, out RaycastHit hit, rayDistance, terrainLayerMask))
            continue;

        bool isDuplicated = false;
        for (int i = 0; i < hitPositions.Count; i++)
        {
            if ((hitPositions[i] - hit.point).sqrMagnitude < minSqr)
            {
                isDuplicated = true;
                break;
            }
        }

        if (isDuplicated)
            continue;

        hitPositions.Add(hit.point);

        GameObject landingObj = Instantiate(landingPlaftormObj, hit.point, Quaternion.identity, transform);

        if (landingObj.TryGetComponent(out LandingPlatform landingPlatform))
        {
            landingPlatform.birdMovement = birdMovement;
            landingPlatform.crowdSpawning = spawning;
        }

        landingObj.name = $"Platform {hitPositions.Count - 1}";
        platformList.Add(landingObj);
    }
}
```

Then I let the birds land on the building

![ForPResent2](https://github.com/user-attachments/assets/a0b75d89-9c15-4cfb-9067-64387e96d5dc)

To do this I have a coroutine chains

```c#
    public IEnumerator FlyToALandingPos(Vector3 target)
    {
        const float approachStopDistance = 3f;
        const float snapDistance = 0.1f;

        const float approachSpeed = 10f;
        const float landingSpeed = 3f;

        float approachStopSqr = approachStopDistance * approachStopDistance;
        float snapSqr = snapDistance * snapDistance;

        while ((transform.position - target).sqrMagnitude > approachStopSqr)
        {
            FlyTo(target, approachSpeed);
            yield return null;
        }

        if (autoAnimate)
        {
            howMnaytimesItCalledLanding++;
            playAnimationComponent.PlayAnimation(AnimationType.landing, true);
        }

        Vector3 finalEuler = transform.eulerAngles;
        finalEuler.x = 0f;

        while ((transform.position - target).sqrMagnitude > snapSqr)
        {
            FlyTo(target, landingSpeed, false);
            yield return null;
        }

        transform.eulerAngles = finalEuler;
        transform.position = target;

        if (waitingInIdleCoroutine == null)
            waitingInIdleCoroutine = StartCoroutine(WaitingInIdle(waitingOnGroundTime));
    }
    
```

```c#
public IEnumerator WaitingInIdle(float waitSeconds = 2f)
{
    // Stop flapping while idle
    if (flappingAnimationCoroutine != null)
    {
        StopCoroutine(flappingAnimationCoroutine);
        flappingAnimationCoroutine = null;
    }

    if (autoAnimate)
    {
        // Pick one of 3 idle animations
        AnimationType[] idles = { AnimationType.idle1, AnimationType.idle2, AnimationType.idle3 };
        int index = Random.Range(0, idles.Length);
        playAnimationComponent.PlayAnimation(idles[index], true);
    }

    yield return new WaitForSeconds(waitSeconds);

    if (!isScared && LeaveThePlatformCoroutine == null)
    {
        LeaveThePlatformCoroutine = StartCoroutine(LeaveThePlatform());
    }
}
```
```c#
    public IEnumerator LeaveThePlatform()
    {
        if (landingPlatfrom == null)
            yield break;

        Transform platformTf = landingPlatfrom.transform;
        Vector3 platformPos = platformTf.position;

        if (landingPlatfrom.TryGetComponent(out LandingPlatform platformComp))
            platformComp.test = true;

        Vector3 takeoffTarget = platformPos + new Vector3(10f, 10.3f, 0f);

        if (flappingAnimationCoroutine != null)
        {
            StopCoroutine(flappingAnimationCoroutine);
            flappingAnimationCoroutine = null;
        }

        if (autoAnimate)
            playAnimationComponent.PlayAnimation(AnimationType.takeOff, true);

        const float arriveDistance = 0.5f;
        float arriveSqr = arriveDistance * arriveDistance;

        const float targetSpeed = 5f;
        const float accelDuration = 1.0f; // seconds to reach targetSpeed (tweak)
        float t = 0f;

        // Accelerate and move until close enough
        while ((transform.position - takeoffTarget).sqrMagnitude > arriveSqr)
        {
            t += Time.deltaTime / accelDuration;
            float currentSpeed = Mathf.Lerp(0f, targetSpeed, Mathf.Clamp01(t));

            FlyTo(takeoffTarget, currentSpeed, true);
            yield return null;
        }

        // Snap to target
        transform.position = takeoffTarget;

        if (flyInAirCor == null)
            flyInAirCor = StartCoroutine(FlyToRandom(true, targetSpeed));

        if (resettingCoroutine == null)
            resettingCoroutine = StartCoroutine(ResetWithDelay(delayForResetBird));

        LeaveThePlatformCoroutine = null;
    }
```
