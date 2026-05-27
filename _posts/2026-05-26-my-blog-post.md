## A Tutorial on Terrain Deformation
by Sam Francis

<!-- insert gif of terrain being deformed -->

![IntroGif](/assets/dig_01.gif)

# __Introduction__

There are two important parts to this topic:
Having a 3d terrain generator (in this case we are using a marching cube generator)
A way to deform/ modify the existing mesh

As this topic is fairly expansive, we will be focusing on the deformation rather than the terrain generation, however I will still go over my implementation briefly. The idea for the deformation would (ideally) work for any 3D terrain generator, that being one that uses three inputs to generate terrain, unlike heightmaps (which only use two inputs). That being said, the concepts from this could be reworked to fit with other dimensions.

![CycleTerrain](/assets/godotPro3_02.gif)

# __A Brief Walkthrough on Generating Terrain.__

For my first attempt at creating a marching cube generator, I had followed a tutorial on youtube titled “Voxel Terrain” by the youtuber Timegame (https://www.youtube.com/watch?v=4lsva1_OTjc), this demonstrates a visual terrain generator tool in Godot that can regenerate terrain in the editor. This would create a single chunk of geometry that could be adjusted based on different parameters, such a noise size, threshold and random seed. This was a good first step at understanding how marching cubes worked, but it wasn’t amazingly performant or scalable: changing any of the values would completely regenerate the mesh.

However, it shows us the basic flow of how the terrain generator works:

1. Take in input variables such as size, noise threshold, seed, etc.
2. Create arrays for variables we want to use later (such as vertex position, colour and normals)
3. Iterate through a x, y and z for loop.
4. Sample 3d noise using a vector3 of the x, y, z positions -> this is maybe the most important step as it provides the density of a given point
5. Feed this position into other functions and look up tables to retrieve data such as corner positions.
6. This will also look at a table of possible shapes for the voxel to be, meaning the voxel “cubes” look smoother than a game like Minecrafts, deliberately blocky, terrain.
7. Store all of this data in the created arrays and build the mesh by passing the relevant arrays into the right spots. For example, I used an array mesh 3d in my implementation as I was able to pass in all of the values without much thought (and it helped with scalability) 

![TerrainCubes](/assets/godotPro3_01.gif)

As you can imagine, the actual process is a lot longer, and not necessarily exciting. This is all to say that the approach itself isn’t that important, however, points 4 and 5 (where the position is sampled) will be helpful when it comes to deforming this newly created terrain. Essentially, to make the terrain… terrain, we have to say what parts are ‘filled in’ and what parts are empty, in this approach we have an arbitrary number that acts as our threshold and if the sampled noise (point 4) is below that threshold then we tell our script that it will be empty. For example, we go through our loops, our x, y, z position looks like Vector3(8,9,10), we now sample the 3D noise map that we created before the loop with this specific position, it has a value of 0.4 which is below our Threshold of 0.6 so we tell it to be empty space.

So what does this have to do with Terrain Deformation?

# __Deforming Terrain__

So, you have your terrain script and its generating nice, boring 3d geometry. How do you add your own flair to that?
My approach for this is to create an empty 3D ‘texture’ (in my case I used a dictionary) that the player can modify in game, this texture can then be seen by the terrain generator and the values from that are added. We are going to be going to where our Threshold value is affecting stuff and say to also use our new 3D texture. Imagine that you have pointed at a section of the terrain and clicked, we want to get that position and give it to our new texture to change the value on that. From there, we go back through the terrain generation system and we look at the value of the terrain in that specific spot and then change it accordingly. For example, we click down whilst looking at a wall, the wall's value was 0.8 but our texture’s value is now -1 at that same position, if we combine the two values we can effectively dig through this wall.

Let’s go through my implementation step-by-step:

![AddingTerrain](/assets/add_01.gif)

__The Player Script__

This will probably be where there's the largest gap in approaches. Maybe you want to mine and dig through the ground, maybe explosive projectiles deform the earth, maybe terraforming machines add mountains to the skyline, etc. For my approach, the player just has to look at a point and click to either add or remove terrain.

```
func test_dig(digging: bool = true):
	
	var hit = edit_cast.get_collider()
	var hit_pos = edit_cast.get_collision_point()
	if not is_instance_valid(hit): return
	
	var tparent = hit.get_parent().get_parent()
	if not is_instance_valid(tparent): return
	if tparent.is_in_group("Terrain"):
		var dist = (hit_pos - global_position).length()
		print(str(dist))
		
		
		if digging:
			tparent.terraform(hit_pos, -dig_strength, dig_size)
			
			var dig_effect = dig_fx.instantiate()
			get_tree().root.add_child(dig_effect)
			dig_effect.global_position = hit_pos
		elif dist > 1:
			tparent.terraform(hit_pos, dig_strength, dig_size)
			
			var dig_effect = dig_fx.instantiate()
			get_tree().root.add_child(dig_effect)
			dig_effect.global_position = hit_pos
```

So going through it, we are referencing our raycast 3d component (however in other engines you could just write a raycast query). We use this to see if the object we are looking at is terrain that we can edit and to save the position of where our ray hits the terrain.
Then based on if we are adding or subtracting terrain, we will call a function on the terrain script and pass in the position that we just received.

__The Terraforming Function__

```
func terraform(world_pos: Vector3, strength: float, brush_size: int):
	var center : Vector3i = grid_space(world_pos, global_position)
	var radius : int = brush_size
	var chunk_offset = (chunk_coord * (size * 2 * resolution))
	var effected_chunks := {}
	var hit_neighbours : Array[Vector3i] #testing
	var times_looped: int = 0 # for testing
	
	for x in range(center.x - radius, center.x + radius):
		for y in range(center.y - radius, center.y + radius):
			for z in range(center.z - radius, center.z + radius):
				times_looped += 1
				var local_p = Vector3i(x, y, z)
				var global_p = local_p + chunk_offset
				
				var chunk_p = find_chunk_pos(global_p)
				if chunk_p != chunk_coord and !hit_neighbours.has(chunk_p):
					hit_neighbours.append(chunk_p)
				#print("#" + str(times_looped) + ": hit chunk: " + str(chunk_p) + " my chunk: " + str(chunk_coord))
				
				var dist = Vector3(x, y, z).distance_to(center)
				if dist > radius: continue
				
				var falloff = 1 - (dist / radius)
				var cur_val = ChunkManager.global_deform_dict.get(global_p, 0.0)
				cur_val = clampf(strength * falloff, -1, 1) 
				
				var effected_chunk = voxel_to_chunk(global_p)
				effected_chunks[effected_chunk] = true
				
				#deform_dict[local_p] = cur_val
				ChunkManager.global_deform_dict[global_p] = clampf(cur_val, -1.0, 1.0)
	gen_on_new_thread()
```

Going over our new Terraform function, we are just taking in the position we found before from the player’s raycast, and then translating that into something that our 3D dictionary can take in (the ‘empty’ texture). The falloff and brushsize are only being used for setting how ‘wide’ and smooth our deformations are, whilst strength determines how much terrain we are adding or subtracting. 
So in this step we are just: taking in the player’s values > converting them to ‘local’ space > adding those values to the ‘texture’ > telling the terrain to regenerate.

Ah but what is that at the end? The whole terrain is regenerating when an edit is made? Didn’t I say that that is a bad idea?
Yes and yes. It is unideal for a final implementation as it would be less performant for a larger scale system, however I am using multiple smaller chunks and for demonstrative purposes this is fine.

Moving on, it is important to look at the ‘ChunkManager.global_deform_dict’. This is just the blank texture I had mentioned before, with slight adjustments to work with multiple chunks. I have made it global so we can pass in a value (and key) from anywhere, allowing us to work between chunks (as the dictionary isn’t local to each chunk). However, for the sake of learning, this could just be a simple Vector 3 int dictionary that is local to a single chunk and everything would work the same (within the chunk).

So now we know that we are editing this dictionary, but where is it being used? What is it? Where did it come from?

__The Dictionary__

```
func init_grid():
	if not is_inside_tree(): return
	
	var start = -float(size) * float(resolution)
	var end = float(size) * float(resolution)
	
	for x in range(start, end):
		for y in range(start, end):
			for z in range(start, end):
				var center := Vector3i(x,y,z)
				var deform_val = 0.0
				var chunk_pos = center + (chunk_coord * (size * 2 * resolution))
				
				if y + global_position.y <= floor_height: #make sure this only happens once to avoid regenerating floor
						deform_val += fill_amount
				
				#clamp for safety
				deform_val = clampf(deform_val, -1.0, 1.0)
				
				deform_dict[center] = deform_val #local
				ChunkManager.global_deform_dict[chunk_pos] = deform_val #global on chunk manager
```
In this function (that is called once at the start of the scene) we go through our for loop that’s the same size as our terrain’s size, and set the dictionary’s values to their default ones. Here we are also setting a certain amount of them to 1 (or fill_amount) to turn them into the ground, however for the sake of deformation, this can be skipped.

`var cube_values: Array[float] = get_cube_value(noise, cube_vertices, local_center, origin)`

Inside the terrain generation loop, in the section that determines the density of the voxel (in our case ‘get_cube_value’) and therefore how it is generated. We will insert all of our deformation code. Meaning that we don’t have to look for the deformation code in multiple areas, it is just in the initialising function ‘init_grid’ (where we set everything to default values), in the terraforming function, and in the function where we determine the density. For your implementation I would recommend keeping everything concise for the sake of troubleshooting.

```
func get_cube_value(noise: FastNoiseLite, cube_vertices: Array[Vector3], c : Vector3i, origin: Vector3) -> Array[float]:
	var values : Array[float] = []
	
	for i in range(8):
		var world_v = cube_vertices[i] + origin
		var noise_val = noise.get_noise_3d(world_v.x, world_v.y, world_v.z)
		var voxel_pos = c + CORNER_OFFSETS[i]
		var global_voxel = voxel_pos + (chunk_coord * (size * 2 * resolution))
		noise_val += ChunkManager.global_deform_dict.get(global_voxel, 0.0) #mesh adds deforms based on THIS
		#moved floor stuff to initial generation
		values.append(clampf(noise_val, -1, 1))
	
	return values
```

Inside this function we are converting each cube vertex (the corners) to a global position, using the global position to sample our 3D noise, doing the same for the cubes center (voxel_pos) and then, most importantly for the deformation, we are using the global center position as the key for our deformation dictionary (ChunkManager.global_deform_dict) to return a value. This approach means that it will add nothing to the terrain if we haven’t terraformed there, and if we have something stored there, it will add our changes to the terrain.

And that is it. Mostly. The rest of this script is wholly dependent on your intentions and can be changed however you see fit. Going back over everything, once you have a way to generate 3D terrain, you then want to:

Have a way to communicate to the terrain script (we use a raycast on a player object)
Have a ‘terraform’ function that controls how the terrain will be modified (i.e. brush size) and updates the changes to a dictionary or other applicable variable
Sample the dictionary before creating the mesh/ adding the vertices to see if terrain should be added or removed.

In terms of simple implementation, this will get you most of the way there, however it is important to keep in mind other aspects that may cause problems.

# __Limitations__

A common problem with terrain deformation is chunks, or more specifically chunk seams. If I dig near the seam of one chunk then how do I tell the connected one to update? My approach for this works but is not scalable: I update all 26 neighbouring chunks…In this project my performance sins may be forgiven as this guarantees that all affected chunks get updated, however it would be smarter to find a way to check the neighbouring chunks of the affected voxel rather than chunk.

In relation to chunks, it is also not unlikely that you may end up with problems of the deformation data not working across different chunks. However, in my case this is why I decided to use a static global dictionary, as all of my chunks can use the same resource.

Another limitation is collisions. Not only are concave colliders a universal no-no in game engines (from a performance point of view), but if you are standing too close to a certain point then you may fall through the ground if the collider generates around you. This can be fixed with distance checking the raycast when the player presses the dig or add buttons.

![Falling](/assets/fall_01.gif)

# __Conclusion__
This project is far from market ready, however it has been fun trying to learn different approaches to terrain deformation, especially in an engine with less tutorials/ resources available for less experienced developers. In the spirit of Godot, I will also be making this project (as it currently is) publicly available and would encourage you to reach out to me if you had any suggestions or recommendations on where you could see this going.

Many thanks and good luck with your projects,

Sam Francis.


