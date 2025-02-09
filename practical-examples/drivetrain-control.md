---
description: >-
  Drivetrains are arguably the most important part of a first tech challenge
  autonomous and are crucial to get correct.
---

# Drivetrain Control

## Why is Drivetrain Control Hard?

In FTC we often want to drive our robot to a desired x, y, and angle position on the field.  This requires two equally important things:

* A way to measure your robots position
* A way to move your robot to a desired position&#x20;

In recent years, the idea of odometry has become increasingly popular in First Tech Challenge. Odometry simply means that we are using sensors to observe the position of your robots drive train.  The most common method of odometry in FTC is using encoders either on the drive wheels of your robot or with seperate sprung wheels known as [dead wheels.  ](https://gm0.org/en/latest/docs/robot-design/dead-wheels.html)There exists several popular libraries for doing the position estimation math but the two that we recommend are the three wheel localizer in the [Roadrunner quick start](https://github.com/acmerobotics/road-runner-quickstart) and [HolonomicOdometry](https://github.com/FTCLib/FTCLib/blob/master/core/src/main/java/com/arcrobotics/ftclib/kinematics/HolonomicOdometry.java) in FTCLib.&#x20;

Once you have a method of robot localization picked out we can begin deciding how we want to control our robot. &#x20;

Traditionally in FTC, teams used their drive encoders to drive straight some distance, switch to using the gyro to turn, and repeat for the duration of their drivetrain control. While this method is simple and effective, it is not necessarily the most robust solution. Modern FTC control systems rely on a concept known as pose stabilization. Pose stabilization uses a single controller to move each axis of your system (x,y, theta) to zero error.

## Mecanum Drivetrain Controller&#x20;

To start with controlling a mecanum drivetrain we first need for figure out how to control each robot relative axis.  A quick look at the [game manual 0 mecanum programming](https://gm0.org/en/latest/docs/software/mecanum-drive.html) guide details that given an x,y, and theta control input we can control a mecanum robot like the following:&#x20;

```java
// x, y, theta input mixing
frontLeftMotor.setPower(x + y + t);
backLeftMotor.setPower(x - y + t);
frontRightMotor.setPower(x - y - t);
backRightMotor.setPower(x + y - t);
```

The code above will allow us to very easily move our robot in the coordinate space relative to itself. &#x20;

In order to move our robot we need to [rotate](https://en.wikipedia.org/wiki/Rotation\_matrix) the x and y commands by the robots angle:

![Visual representation of rotating the robot relative vector into the field relative space](../.gitbook/assets/rotated-mecanum-controller-.png)

To perform this rotation we can take our x command, y command, and the angle of our robot and use the rotation equation to solve for the rotated commands

```java
double x_rotated = x * Math.cos(angle) - y * Math.sin(angle);
double y_rotated = x * Math.sin(angle) + y * Math.cos(angle);
```

{% hint style="success" %}
FTClib users can accomplish this built in by using Vector2d.rotateby(angle) method.&#x20;
{% endhint %}

we can then combine this rotation with our mecanum control mixing to get the following:&#x20;

```java
// rotation 
double x_rotated = x * Math.cos(angle) - y * Math.sin(angle);
double y_rotated = x * Math.sin(angle) + y * Math.cos(angle);

// x, y, theta input mixing
frontLeftMotor.setPower(x_rotated + y_rotated + t);
backLeftMotor.setPower(x_rotated - y_rotated + t);
frontRightMotor.setPower(x_rotated - y_rotated - t);
backRightMotor.setPower(x_rotated + y_rotated - t);
```

{% hint style="success" %}
If one was to use gamepad joystick inputs for x,y,t they can use this code above to accomplish field relative driving!
{% endhint %}

We are at the point now that given any input, x,y,t it will push our robot in the direction regardless of the angle, we can even combine multiple inputs such as moving along diagonal paths while rotating.  We have reached a state where our system is now fully actuated and nearly completely linearized.  This means from this point on our system is now relatively trivial to control.&#x20;

For getting the values of x,y, and t we can simply use a linear controller such as the full state feedback or PID controllers that we developed in previous chapters&#x20;

```java
xControl = new PID();
yControl = new PID();
thetaControl = new PID();

while (loopIsActive) {
    // this imaginary pid controller has a control method that uses the 
    // PID controller we defined earlier in a method called calculate
    // the first argument of calculate is the reference state. 
    // the second is the systems state. 
    x = xControl.calculate(xTarget, xRobotPosition); 
    y = yControl.calculate(yTarget, yRobotPosition);
    t = thetaControl.calculate(thetaTarget, thetaRobotPosition);
    double x_rotated = x * Math.cos(angle) - y * Math.sin(angle);
    double y_rotated = x * Math.sin(angle) + y * Math.cos(angle);
    
    // x, y, theta input mixing
    frontLeftMotor.setPower(x_rotated + y_rotated + t);
    backLeftMotor.setPower(x_rotated - y_rotated + t);
    frontRightMotor.setPower(x_rotated - y_rotated - t);
    backRightMotor.setPower(x_rotated + y_rotated - t);
}
```

{% hint style="danger" %}
For thetaControl you will need to consult the [Dealing with Angles](https://www.ctrlaltftc.com/controlling-heading) chapter for changing the way that the angle error calculation is performed.  This will ensure your robot is always turning the shortest distance to the desired angle
{% endhint %}

{% hint style="success" %}
After completing all of these steps you will now have a fully functional pose stabilization controller for a mecanum robot!
{% endhint %}

{% embed url="https://youtu.be/knZLd0yyHw4" %}
Disturbance rejection qualities of similar control technique
{% endembed %}

## Differential Drivetrain Controller

Differential drive robots such as 4 wheel drive, 6 wheel drive, and tank tread robots are unfortunately a beast to control.  This is because of one unfortunate quality that differential drive robots possess: They are underactuated systems. &#x20;

In control theory, an underactuated system is defined as a system that has fewer actuators than they do outputs to control.  Intuitively it makes sense as a fully actuated system such as a mecanum drive can move towards the desired x,y position while turning towards the desired angle.  A differential drive robot cannot do this.  Instead it needs to first turn towards the target x,y position, move towards the target position and then once at the x,y position turn to the desired angle. &#x20;

### Naive Differential Drive Controller&#x20;

While not perfect, this approach to differential drive control can be very effective. This controller works by first driving to the desired position, and then once the robot is within a desired distance to the position turns toward the target angle.&#x20;

This controller requires feedback on three seperate values

* The distance to the target point&#x20;
* The angle to the target point&#x20;
* The reference angle&#x20;

{% hint style="success" %}
Remember: The angle to the target point is the angle between the robot and the target x,y position.  The reference angle is the angle we want the robot to be facing whenever it finishes the movement.&#x20;
{% endhint %}

Using the following will drive the robot at a power proportional to the distance between the robot position and the target position&#x20;

```java
distanceController = new PID();

while (loopIsActive) {
    // 0 is the reference because we want the distance to go to 0 
    double f = distanceController.calculate(0,getDistance(robotX,robotY,tagetX,targetY));
    double t = 0; 
    frontLeftMotor.setPower(f + t);
    backLeftMotor.setPower(f + t);
    frontRightMotor.setPower(f - t);
    backRightMotor.setPower(f - t);
}
```

&#x20;Now we have code that given the robots position and a target position will drive proportional to its remaining distance.  We still need to make the robot turn towards the target point otherwise it will continue on in whatever direction the robot started in forever.

For this, we first need to find a way to calculate the angle between the robot and the target.   Fortunately for us, there is a built in java method to accomplish this known as atan2.  Atan2 is a special trigomentric function that given arguments y,x returns the angle from x,y to the origin.  Since we are getting the angle between two points we use the error between the robot x,y and the target x,y as follows:&#x20;

```java
double xError = targetX - robotX;
double yError = targetY - robotY; 
double theta = Math.atan2(yError,xError);
```

{% hint style="warning" %}
Make sure that you use your y value as the first argument followed by the x value as the java [Math library javadoc](https://docs.oracle.com/javase/8/docs/api/java/lang/Math.html) states is required for the atan2 method to function correctly.&#x20;
{% endhint %}

now that we can calculate this angle we can put this into the control loop for our turn command:&#x20;

```java
distanceController = new PID();
angleController = new PID();  // make sure this follows "Dealing with Angles" 

while (loopIsActive) {
    double xError = targetX - robotX;
    double yError = targetY - robotY; 
    double theta = Math.atan2(yError,xError);
    // 0 is the reference because we want the distance to go to 0 
    double distance = Math.hypot(xError, yError);
    double f = distanceController.calculate(0,distance);
    double t = angleController.calculate(theta, robotTheta); 
    double left_power = f + t;
    double right_power = f - t;
}
```

{% hint style="danger" %}
I understand you are probably tired of hearing this but for angles and feedback control you MUST FOLLOW THE [DEALING WITH ANGLES CHAPTER](https://www.ctrlaltftc.com/controlling-heading) OR YOUR CONTROLLER _**WILL**_ BREAK.&#x20;
{% endhint %}

Now we have a controller that will drive to a desired x,y position on the field but unfortunately will not stop at the desired angle. A quick but unfortunately not very elegant solution is to simply check if we are within an acceptable distance and simply switch to turning the robot:

```java
distanceController = new PID();
angleController = new PID();  // make sure this follows "Dealing with Angles" 

while (loopIsActive) {
    double xError = targetX - robotX;
    double yError = targetY - robotY; 
    double theta = Math.atan2(yError,xError);
    // 0 is the reference because we want the distance to go to 0 
    double distance = Math.hypot(xError, yError);
    double left_power = f + t;
    double right_power = f - t;
    if (distance < threshold) {
         f = 0;
         t = angleController.calculate(targetAngle, robotTheta);      
    }
    // set motor power 
}
```

{% hint style="success" %}
Now you have a controller that will drive to a desired x,y, theta position
{% endhint %}

#### Cosine Trick&#x20;

In order to get better stability, we can slow the drive trains forward movement while the angle error is high.&#x20;

Doing something like the following

$$
p_f=d*k_p*\cos{\theta_e}
$$

​where the power in the forward direction is equal to Kp times the distance as we had before but then an addition cos term is added.  \*Theta\_error will be domain limited to -pi/2 to pi/2\*

![](../.gitbook/assets/image.png)

Looking at the graph of cos shows an interesting curve,  whenever our angle error is high, the value of this cos approaches zero, but as the angle error approaches zero, the cos term goes to 1.   This creates the effect of only allowing our robot to quickly move in the forward direction when it is on target. &#x20;

```java
distanceController = new PID();
angleController = new PID();  // make sure this follows "Dealing with Angles" 

while (loopIsActive) {
    double xError = targetX - robotX;
    double yError = targetY - robotY; 
    double theta = Math.atan2(yError,xError);
    // 0 is the reference because we want the distance to go to 0 
    double distance = Math.hypot(xError, yError);
    double left_power = f + t;
    double right_power = f - t;
    if (distance < threshold) {
         f = 0;
         t = angleController.calculate(targetAngle, robotTheta);      
    } else {
        f = distanceController.calculate(0, distance); 
        t = angleController.calculate(theta, robotTheta);
    }
    // Range.clip is included in the SDK and will clip between two values
    // angleController.error is a demonstrative attribute that gets the error. 
    f *= Math.cos(Range.clip(angleController.error, -PI/2, PI/2));
    
    // set motor power here! 
    setRobotRelative(f,t); 
}
```
