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
