##Drawn to Life

![Drawn to Life](project_images/drawntolife.gif?raw=true "Head Motion")

So now that we have both the animation and drawing part figured out, it's time to put the two together.

Each 'creature' object is simply a THREE.js Object3D with variables stuck on it. 

```
function generateCreature( headMesh, iterations ){
	var creature = new THREE.Object3D();
	creature.timing = Math.random() * Math.PI;
	creature.frequency = controls.frequency;
	creature.amplitude = controls.amplitude;
	creature.steering = Math.random() * Math.PI * 0.001;
	creature.speed = 3.4;
	creature.forward = new THREE.Vector2(0,-1);
	creature.steer = new THREE.Vector2(0,0);
	creature.velocity = new THREE.Vector2(0,0);
	creature.spacing = controls.spacing;
	creature.lifeSpan = Math.random() * 80000 + 8000;		    	
	while( iterations >= 0 ){
		var copiedSegment = copySegment( headMesh, creature.children.length );
		creature.add( copiedSegment );		    				    		
		iterations--;
	}
	return creature;
}
```

This generates the full creature body, and places it as an object in the scene graph which animates. But how does it animate?

##Animation Code Revised
![Playing Hard to Get](project_images/drawnswimming.gif?raw=true "Playing Hard to Get")

Remember our animation code from the previous post? Well that needs some improvements, especially if we want to keep our creature swimming on-screen. Some hacky steering solution here makes the creature eventually want to stay near the middle. Witness some dirty hacks below.

```
function snakeAnimateCreature( creatureObject ){
	var segments = creatureObject.children;
	var head = segments[0];
	creatureObject.speed = 3.4;		    	
	var turning = 0.2;	
	if( Math.random() > 0.99 )
		turning = 3.7;
	if( Math.random() > 0.7 ){
		creatureObject.steer.x -= (head.position.x) - width/2;
		creatureObject.steer.y -= (head.position.y);			    	
		creatureObject.steer.normalize();
		creatureObject.steer.multiplyScalar( turning );
	}
	else{
    	creatureObject.steer.x += (-1 + Math.random() * 2) * turning;
    	creatureObject.steer.y += (-1 + Math.random() * 2) * turning;			    	
    }
	Math.pow(creatureObject.steer.x, 2);
	Math.pow(creatureObject.steer.y, 2);
	creatureObject.steer.multiplyScalar(0.4);

	creatureObject.forward.addSelf( creatureObject.steer );		    			    	
	creatureObject.forward.normalize();		    			    	

	var finalSpeed = creatureObject.speed + Math.pow( 1+creatureObject.steer.length() * 2,2);
	creatureObject.velocity.x += creatureObject.forward.x * finalSpeed;
	creatureObject.velocity.y += creatureObject.forward.y * finalSpeed;
	creatureObject.velocity.multiplyScalar(0.64);

	//	remainder of the code nearly identical

}
```

Note how we now have a 'turning' variable, which tells the creature how far to change steering.

```
var turning = 0.2;	
if( Math.random() > 0.99 )
	turning = 3.7;
if( Math.random() > 0.7 ){
	creatureObject.steer.x -= (head.position.x) - width/2;
	creatureObject.steer.y -= (head.position.y);			    	
	creatureObject.steer.normalize();
	creatureObject.steer.multiplyScalar( turning );
}
else{
	//	...
}
```

Every frame, a dice is rolled and if it's >.99 out of 1, then the creature takes a big turn, otherwise if the dice rolled >0.7 it would stay on course but attempt to steer towards the center of the screen, horizontally. 

In all other cases, the head would steer as it normally would, randomizing between left and right.

Not the most elegant of solutions, but enough to create an illusion of life.


##Wiggle Room

![Wiggle Out](project_images/wiggleout.gif?raw=true "Wiggle Out")

While drawing the creature, it felt a little *dead*, so adding this sinusoidal motion while drawing gave it a sense of urgency, which also acts as a timer to let the user know the thing being drawn is about to escape.

```
if(segments != undefined){
	var freqAdd = (curTime - lastTouchTimed) / 4000;	
 	for( var i=0; i<segments.length; i++ ){
 		var frequency = controls.frequency * (1-freqAdd) * 2;
		var prevSegment = segments[i-1];
		var segment = segments[i];						    		
 		animateSegment( prevSegment, segment, i, frequency, 0.4 + controls.amplitude * (freqAdd * freqAdd) * 1.3 );
 	}
}
```

animateSegment does the hard math of composing the body while it's being drawn

```
function animateSegment( prevSegment, segment, iteration, frequency, amplitude ){
	var y = segment.position.y;
	segment.position.x = Math.sin( frequency + ((iteration+1) * 0.02 + y * 0.02 ) ) * (iteration+1) * amplitude;
	if(iteration <= 0)
		return;
	var x1 = segment.position.x;
	var y1 = segment.position.y;
	var x2 = prevSegment.position.x;
	var y2 = prevSegment.position.y;
	var rotation = Math.atan2( y2 - y1, x2 - x1 ) + turn;
	segment.rotation.z = rotation;			
}
```

Originally I had a 'create' button on the GUI to spawn the creature into existence, but that felt a bit contrived and boring. Letting it actually swim off the drawing canvas seemed way more natural.

