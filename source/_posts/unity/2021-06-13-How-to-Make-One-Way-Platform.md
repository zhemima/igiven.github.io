---
title : "How to Make One Way Platform in unity"
---

Hello Readers, I'll Show you how to make simple one way platform. It is simple, useful and efficient technique to achieve this result. Here is the procedure --

1. Make a Simple platform using Box collider.
2. Make a new game object children of the platform and add a box collider just below the original platform as shown in image and tick its 'is trigger' checkbox so that we can check player's collision in it.

![111](../../assets/images/2021-06-13-How-to-Make-One-Way-Platform/111-1623584452229.jpg)

3. Create a new Script to the game object we create in step 2 and assign the following script --

```c#
//the collider of the main visible platform
var platform : BoxCollider2D;
//this variable is true when the players is just below the platform so that its Box collider can be disabled that will allow the player to pass through the platform
var oneway : boolean;


function Update () {
    //Enabling or Disabling the platform's Box collider to allowing player to pass
    if (oneway)
     platform.enabled=false;
     if (!oneway)
     platform.enabled=true; 
}
//Checking the collison of the gameobject we created in step 2 for checking if the player is just below the platform and nedded to ignore the collison to the platform
function OnTriggerStay2D(other: Collider2D) {
   oneway = true;
}

function OnTriggerExit2D(other: Collider2D) {
//Just to make sure that the platform's Box Collider does not get permantly disabled and it should be enabeled once the player get its through
   oneway = false;
}
```

I've made a simple ball game to illustrate the application of the one way platforms. You can download the example project folder below. --

