##Steering

![Head Motion](project_images/boxmotion.gif?raw=true "Head Motion")

Ah yes the classic [Craig Reynolds](http://www.red3d.com/cwr/steer/gdc99/ "Craig Reynolds") papers come to use here.

Generally speaking, steering is the idea of having an entity derive motion from both it's desired behavior (eg path determination) and how the physics interact with this behavior.

Here's what I've arrived at, with a simple box driving itself around our scene.

```
head.steer.x += (-1 + Math.random() * 2) * 0.2;
head.steer.y += (-1 + Math.random() * 2) * 0.2;
```

This creates, on the head of our creature, a randomized addition to the 'steer' vector. It's just random, there's no noise or anything like that, although that would be an improvement.

Next, we do a bit of vector math:

```
head.steer.normalize();

head.forward.addSelf( head.steer );		    			    	
head.forward.normalize();		    			    	

head.position.x += head.forward.x * head.speed;
head.position.y += head.forward.y * head.speed;
```

This normalizes the steering vector, so that it's always a unit length between 0 and 1. We add this to the forward "facing" vector of the head, and normalize *that* as well, so the desired travel vector is constantly being pulled towards "forward", whilst being modified by "steer".

Finally, this vector is integrated on top of the head position and modified by the speed of this entity. 

This is an ugly hacky solution, but it does the job. Alternatively, we can cause this entity to follow the mouse, which looks something like this

```
var targetX = mouseX - head.position.x;
var targetY = mouseY - head.position.y;
var distToTarget = Math.sqrt( targetX * targetX + targetY * targetY);
if(distToTarget > 10 ){
	head.steer.x = targetX;
	head.steer.y = targetY;
	head.speed = 3.4;		    		
}
else{
	head.speed = 0;									
}
```

To get the head rotated correctly to its path of motion, we do something like this

```
head.rotation.z = Math.atan2( -head.forward.y, -head.forward.x );
```

##Serpentine Motion

I've thought a lot about how to animate the snake-like coil of a body. A lot of choices ended up not being satisfactory, for example, a simple spring-system which generally causes a lot of sharp turns when the movement speed is insufficient. Ultimately, I wanted the creature body to end up feeling like a solid, and maintain some kind of organic rhythm.

![Train of Boxes](project_images/chainlink.gif?raw=true "Train of Boxes")

The solution was two-fold, and it looked something like this:

```
for( var i=0; i<body.children.length; i++ ){
	var segment = body.children[i];
	var prev;
	if(i==0){
		prev = head;
	}
	else{
		prev = body.children[i-1];
	}

	var lenX = segment.position.x - prev.position.x;
	var lenY = segment.position.y - prev.position.y;
	var dist = Math.sqrt( lenX * lenX + lenY * lenY );
	if(dist > 40){
		var direction = new THREE.Vector2(lenX,lenY);
		direction.normalize();
		direction.multiplyScalar(40);
		segment.position.x = prev.position.x + direction.x;
		segment.position.y = prev.position.y + direction.y;
		segment.rotation.z = Math.atan2(lenY, lenX);
	}
	

	var prevVecX = Math.cos( prev.rotation.z );
	var prevVecY = Math.sin( prev.rotation.z );
	var prevVec = new THREE.Vector3(prevVecX, prevVecY);
	var thisVec = new THREE.Vector3(lenX, lenY);
	thisVec.normalize();
	var dotProduct = prevVec.dot( thisVec );
	if( Math.abs(dotProduct) > 0.4){
		prevVec.multiplyScalar(40);		    			
		var targetPos = new THREE.Vector3( prev.position.x, prev.position.y );
		targetPos = targetPos.addSelf(prevVec);		    			
		segment.position.x = segment.position.x + (targetPos.x - segment.position.x) * 0.3;
		segment.position.y = segment.position.y + (targetPos.y - segment.position.y) * 0.3;
	}
}
		    
```

In retrospect I probably shouldn't be instantiating multiple new vector3 objects per frame like this.

Let's follow this though...

```
var lenX = segment.position.x - prev.position.x;
var lenY = segment.position.y - prev.position.y;
var dist = Math.sqrt( lenX * lenX + lenY * lenY );
if(dist > 40){
	var direction = new THREE.Vector2(lenX,lenY);
	direction.normalize();
	direction.multiplyScalar(40);
	segment.position.x = prev.position.x + direction.x;
	segment.position.y = prev.position.y + direction.y;
	segment.rotation.z = Math.atan2(lenY, lenX);
}
```	

The first part, lenX and lenY calculates the distance between the first segment to the next segment of the creature body. We allow 0-40 length of distance so that the creature may appear elastic, and not forced to be rigid.

Past 40 length, we'll constrain it to 40, but maintain the angle that the two joints held. This causes a chain-link behavior, kind of like a train with cars following it.

The next part of the code is slightly more complicated but is the "magic sauce" for motion here.

```
var prevVecX = Math.cos( prev.rotation.z );
var prevVecY = Math.sin( prev.rotation.z );
var prevVec = new THREE.Vector3(prevVecX, prevVecY);
var thisVec = new THREE.Vector3(lenX, lenY);
thisVec.normalize();
var dotProduct = prevVec.dot( thisVec );
if( Math.abs(dotProduct) > 0.4){
	prevVec.multiplyScalar(40);		    			
	var targetPos = new THREE.Vector3( prev.position.x, prev.position.y );
	targetPos = targetPos.addSelf(prevVec);		    			
	segment.position.x = segment.position.x + (targetPos.x - segment.position.x) * 0.3;
	segment.position.y = segment.position.y + (targetPos.y - segment.position.y) * 0.3;
}
```

prevVec calculates the angle at which the previous segment is pointing, so if it's going north it'll be 90 degrees, and west will be 180 degrees (1/2 PI and PI, in radians respectively).

This let's us create a dot product between the previous segment's angle versus the current segment's angle.

[Here's a quick primer / example on dot product](http://www.falstad.com/dotproduct/ "example on dot product")

![Dot Product Example](project_images/dotproduct.gif?raw=true "Dot Product Example")

In the above image we see two vectors, red and blue. On the right we see the dot product between the two. Assuming the two vectors are equal length (eg length of 1 here), we can do some fun things like angle comparisons of the two vectors. So for example, when the two vectors are pointing the same direction, their dot product is 1.0. When the two are perpendicular, the dot product is 0.0 (either direction). And when they are pointing away from each other, the dot product is -1.0.

In my code above, we calculate the dot product between the angle of the previous segment versus the current segment of the tail during the loop. Here, an if statement checks to see if the dot product is greater than 0.4 which is about 66 degrees. Meaning, if any two segments differ in their facing angles by 66 degrees, it will execute the following code.

```
if( Math.abs(dotProduct) > 0.4){
	prevVec.multiplyScalar(40);		    			
	var targetPos = new THREE.Vector3( prev.position.x, prev.position.y );
	targetPos = targetPos.addSelf(prevVec);		    			
	segment.position.x = segment.position.x + (targetPos.x - segment.position.x) * 0.3;
	segment.position.y = segment.position.y + (targetPos.y - segment.position.y) * 0.3;
}
```	

Here, we see that I'm creating a "targetPos", meaning this is where the segment "should be", given this angle difference.

```
segment.position.x = segment.position.x + (targetPos.x - segment.position.x) * 0.3;
```

This particular line 'interpolates' the segment to the target position, 30% every frame this code is run.

Some interesting emergent behavior happens here. The head is receiving random steering changes via our steering code, but the body is experiencing this self-correction as if to simulate an elastic body trying to maintain a straight line. This behavior conflicts with the position correction via distance (the train-cars effect) and you get this wiggling snake-like behavior.

Witness the magic:

![Serpentine](project_images/serpentine.gif?raw=true "Serpentine")

You can play with the live version here:
[https://dl.dropboxusercontent.com/u/705999/metamorph/serpentine.html](https://dl.dropboxusercontent.com/u/705999/metamorph/serpentine.html)

Hold mouse button to manually steer the creature. Notice how un-life-like it is with just its tail moving about, compared to the randomized motion given to its head.

That's all for today! I've got two more updates to make and we'll be off to the publish button.