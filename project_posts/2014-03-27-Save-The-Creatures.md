## The Cloud
Before we wrap up, there is one final, critical step to all of this.

We need to save and load.

The platform I chose for this was Google's Appengine, which has storage. I have no idea what my quota was, and my fetch operations seemed really clumsy and I may or may not run out of quota so this might totally fail when it goes public. But here goes.

## Data Format
Remember a few posts back how I insisted on storing just the drawing vertices? This is going to save us here, because we'll be storing the creature's genotype as penstrokes as well as DAT.GUI configuration.

![Simple Drawing](project_images/simpledrawing.png?raw=true "Simple Drawing")

First, the penstrokes:

```
[[{"x":-2,"y":0},{"x":-5,"y":3},{"x":-9,"y":7},{"x":-11,"y":9},{"x":-16,"y":11},{"x":-18,"y":13},{"x":-20,"y":13},{"x":-21,"y":15},{"x":-23,"y":15},{"x":-23,"y":17},{"x":-25,"y":18},{"x":-26,"y":20},{"x":-27,"y":22},{"x":-29,"y":23},{"x":-29,"y":26},{"x":-31,"y":28},{"x":-31,"y":30},{"x":-33,"y":31},{"x":-33,"y":33},{"x":-34,"y":35},{"x":-36,"y":37},{"x":-37,"y":39},{"x":-37,"y":41},{"x":-39,"y":43},{"x":-39,"y":45},{"x":-39,"y":47},{"x":-39,"y":49},{"x":-39,"y":51},{"x":-37,"y":52},{"x":-35,"y":53},{"x":-33,"y":54},{"x":-31,"y":54},{"x":-28,"y":54},{"x":-25,"y":54},{"x":-23,"y":54},{"x":-20,"y":53},{"x":-17,"y":51},{"x":-14,"y":51},{"x":-12,"y":50},{"x":-10,"y":50},{"x":-8,"y":51},{"x":-8,"y":53},{"x":-6,"y":53},{"x":-4,"y":53},{"x":-2,"y":54},{"x":0,"y":55}]]
```

Okay, this is not the world's best way to pack vertices. Anyway, let's stringify this and put it on the cloud.

```
function saveCreature( meshJSON, callback ){
	var meshData = { "mesh": JSON.stringify(meshJSON) };

	$.post('api/store_creature', meshData, function() {
		console.log('posted');
		console.log(meshData);
	});						    	
}
```

What is api/store_creature? This is some python written on Appengine side to handle the storage. It looks like this.

```
class Creature( db.Model ):
	mesh = db.TextProperty()

class CreatureStore( webapp.RequestHandler ):
	def post(self):
		creature = Creature()
		creature.mesh = self.request.get("mesh")
		creature.put()
		logging.debug( self.request.get("mesh") )
```

What about loading? Well, which creature are we going to load? This gets rather nasty, and I hope I learn something out of all of this. On server end, it'll look like this:

```
class LoadRandom( webapp.RequestHandler ):
	def get(self):
		creatures = Creature.all()
		for creature in creatures:			
			logging.debug( creature.mesh )

		randomIndex = random.random() * creatures.count()
		randomIndex = int( math.floor( randomIndex ) )
		
		# is there a better way?
		results = creatures.fetch( offset=randomIndex, limit=1 )
		for result in results:
			self.response.out.write( result.mesh )		
```

This gets us a random index from the list of all creatures, and then does a fetch. From what little I know, this seems pretty expensive, so maybe someone can educate me on the subject.

On the client side, we essentially have to 'rebuild' the creature via the saved pen-stroke data.

```
function loadRandomCreature( count ){	
	$.get('api/load_random', function(results){
		// console.log(JSON.parse(results));
		var meshStrokes = JSON.parse(results);
		var mesh = new THREE.Object3D();
		
		for( var i=0; i<meshStrokes.length; i++ ){
			var stroke = meshStrokes[i];
			var geo = new THREE.Geometry();	
			var mirroredGeo = new THREE.Geometry();
			for( var s=0; s<stroke.length; s++){
				var p = stroke[s];
				var vec = new THREE.Vector3(p.x, p.y, 0);
				var vert = new THREE.Vertex(vec);
				geo.vertices.push(vert);

				var mirroredVert = new THREE.Vertex( new THREE.Vector3(-p.x, p.y, 0) );
				mirroredGeo.vertices.push( mirroredVert );

			}
			var strokeMesh = new THREE.Line(geo, penMat, THREE.LineStrip);
			mesh.add(strokeMesh);
			var mirroredMesh = new THREE.Line(mirroredGeo, penMat, THREE.LineStrip);
			mesh.add(mirroredMesh);
		}
		generateCreatureVariant(mesh, controls.segments);
	});
}
```

Note the last line. Because we're dynamically generating these creatures, we can actually make subtle variations of them, simply by modifying the number of segments and anything else we can tune.

So something like...

```
var generateCreatureVariant = function( drawnMesh, segmentCount ){
	var size = 0.5 + Math.random() * 0.5;

	var segments = segmentCount - 10 + 16 * size;
	segments = Math.floor(segments);
	if( segments < 3 )
		segments = 3;

	var creature = generateCreature( drawnMesh, segments );
	ocean.add(creature);

	creature.position.x = Math.random() * width * 0.4 - width * 0.2;
	creature.position.y = Math.random() * height * 0.3 - height * 0.15 + height/2;
	creature.rotation.z = Math.random() * 360 * Math.PI / 180 + Math.PI/2;					

	creature.scale.x = creature.scale.y = 0.05 + size * 0.4;
}
```

And finally we've added a creature that's loaded from AppEngine to a random location in our world. 

## The Ocean
So now that we can save and load, I've created an AppEngine instance of this project that saves every creature ever drawn. (It's most likely over quota by the time you see it.) By visiting the page it will pull a single creature and spawn 3 of them, and invite you to draw your own.

The variations are quite astounding!

![Swimming](project_images/swimming.gif?raw=true "Swimming")

![Swimming](project_images/swimming2.gif?raw=true "Swimming")

![Swimming](project_images/swimming3.gif?raw=true "Swimming")

I'll post the remainder on an [imgur album](http://imgur.com/a/wcAvH) as well as in the main content area above.

You can draw your own at [http://drawcreature.appspot.com/](http://drawcreature.appspot.com/)

## Future Work
Aside from the obvious technical improvements, such as data format and save/load inefficiencies, there are some areas which I think can be thoroughly improved.

The motion of the organism all remain to be about the same, they're basically the same creatures with different skin. It would be interesting to apply some kind of physics to the drawn lines so that the shapes can bend and stretch. That would 'automatically' make the drawings themselves more meaningful. 

Another area where I totally avoided was doing some kind of fluid field dynamic so that the organism could create waves and push each other around. This begins to become a larger question of how to house these creatures, and how to show-case them in interesting ways.

Finally, it's tempting to start adding bioluminescent lights and such however I really do enjoy the level of simplicity this project has, visual and design both. I even avoided anti-aliasing to keep the aesthetic as raw as it could be, so "it is what it is". Perhaps there's room for a more visually vibrant version. That's for another day perhaps.