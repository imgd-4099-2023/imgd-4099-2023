# Towards Boids on the GPU

This tutorial aims to get us at least part of the way towards implementing 
the [Boids (aka Flocking) algorithm](http://www.kfish.org/boids/pseudocode.html), 
originally presented [in a classic paper that has been cited over 11000 times](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=3&cad=rja&uact=8&ved=2ahUKEwiqy5OP_tPoAhWDuJ4KHdqmDH4QFjACegQIIhAB&url=http%3A%2F%2Fwww.cs.toronto.edu%2F~dt%2Fsiggraph97-course%2Fcwr87%2F&usg=AOvVaw2StOMrXs0E_nHLgD87UrGN). 
[Here’s a video of the original simulation in action](https://www.youtube.com/watch?v=86iQiV3-3IA). 
For an in-depth treatment of agent-based steering algorithms more generally (including Boids) 
try [this chapter from the Nature of Code](https://natureofcode.com/book/chapter-6-autonomous-agents/).  T
he author of this chapter also walks through implementing the algorithm in 
[his amazing Coding Train series](https://www.youtube.com/watch?v=mhjuuHl6qHM). 
But the first link in this assignment is perhaps the easiest to follow, with clear pseudocode
and explanations of the algorithm. 

In some ways, this simulation is simpler than our previous Physarum simulation, as we don't have an
underlying continuous layer to deal with (unless you want to add one that influences the agents in some way).
But our complexity now becomes that boids need to know about each other in a way we haven't explored before.
In the game of life, cells needed to know about their immediate neighbors; 
here every agent needs to (potentially) know about every other agent in the flock.

This means we need a way to loop through the flock. A `for` loop is all we need for this.

Remember that if, for every given agent A, we're looking up all other agents stored in group A\*
of size 8192, that means processing all agents will require 8192*8192 buffer lookups... 67 million!
And that's not even including all the actual vector math that is performed once 
we have the position of each agent. That said, on even a moderately new integrated graphics card
you can expect a 15-20x speedup by running boids on the GPU instead of the CPU. 
For fancy graphics cards (1080, 2080, 3060 on up) you can expect even more than this; definitely worth
the effort.

OK, here we go!

## The Setup

The [template project we'll use for this tutorial](https://github.com/charlieroberts/seagulls/tree/main/demos/14_many_particles) is the seagulls `many_particles` demo. 
We don't need to do anything to our `main.js` file or our `render.glsl` file; everything is all set to go there.
We just need to modify our compute shader. Once boids is up and running, you're welcome to play around with
the rendering / number of agents if you like.

Let's go ahead and open up our `compute.glsl` file. Make sure you change the `Particle` definition at the top of
file to use a `vec2f` for velocity instead of a single float for `speed`.

```wgsl
struct Particle {
  pos: vec2f,
  vel: vec2f
};

@group(0) @binding(0) var<uniform> frame: f32;
@group(0) @binding(1) var<uniform> res:   vec2f;
@group(0) @binding(2) var<storage, read_write> state: array<Particle>;

fn cellindex( cell:vec3u ) -> u32 {
  let size = 8u; // 8 is our workgroup size on one axis
  return cell.x + (cell.y * size);
}

@compute
@workgroup_size(8,8,1)

fn cs(@builtin(global_invocation_id) cell:vec3u)  {
  let count:u32 = 2048u;

  let idx            = cellindex( cell );
  var boid:Particle  = state[ idx ];

  var center:vec2f   = vec2f(0.); // rule 1

  for( var i:u32 = 0u; i < count; i = i + 1u ) {
    // don't use boids' own properties in calculations
    if( idx == i ) { continue; }

    let _boid = state[ i ];
    
    // rule 1
    center += _boid.pos;
  }

  // apply effects of rule 1
  center /= f32( count - 1u );
  boid.vel += (center-boid.pos) * .05;
  
  // calculate next position
  boid.pos = boid.pos + (2. / res) * boid.vel;

  // boundaries, flip velocity if you pass one
  if( abs( boid.pos.y ) >= 1. ) { 
    boid.vel.y *= -1.; 
  }
  if( abs( boid.pos.x ) > 1. ) {
    boid.vel.x *= -1.;
  }

  // make sure to reassign back to buffer
  // otherwise no effect will occur!
  state[ idx ] = boid;
}
```

In the above code we implement rule 1 for boids, which tells each boid to move towards 
a center position by averaging the position of all other boids and using this value to
influence the boids velocity. We also invert the velocity whenever we cross a border (-1 or 1)
on the x and y axes.

It's the same basic process for rules two and three, just with a bit different math. Rule 2 will 
tell each boid to try and keep a minimum distance from other boids, while Rule 3 will tell boids
to try and match velocities.

```wgsl
struct Particle {
  pos: vec2f,
  vel: vec2f
};

@group(0) @binding(0) var<uniform> frame: f32;
@group(0) @binding(1) var<uniform> res:   vec2f;
@group(0) @binding(2) var<storage, read_write> state: array<Particle>;

fn cellindex( cell:vec3u ) -> u32 {
  let size = 8u;
  return cell.x + (cell.y * size) + (cell.z * size * size);
}

@compute
@workgroup_size(8,8)

fn cs(@builtin(global_invocation_id) cell:vec3u)  {
  let count:u32 = 2048u;

  let idx            = cellindex( cell );
  var boid:Particle  = state[ idx ];

  var center:vec2f   = vec2f(0.); // rule 1
  var keepaway:vec2f = vec2f(0.); // rule 2
  var vel:vec2f      = vec2f(0.); // rule 3

  for( var i:u32 = 0u; i < count; i = i + 1u ) {
    // don't use boids' own properties in calculations
    if( idx == i ) { continue; }

    let _boid = state[ i ];
    
    // rule 1
    center += _boid.pos;
    
    // rule 2
    if( length( _boid.pos - boid.pos ) < .005 ) {
      keepaway = keepaway - ( _boid.pos - boid.pos );
    }
    
    // rule 3
    vel += _boid.vel;
  }

  // apply effects of rule 1
  center /= f32( count - 1u );
  boid.vel += (center-boid.pos) * .05;

  // apply effects of rule 2
  boid.vel += keepaway;

  // apply effects of rule 3
  vel /= f32( count - 1u );
  boid.vel += vel * .01;
  
  // calculate next position
  boid.pos = boid.pos + (2. / res) * boid.vel;

  // boundaries
  if( abs( boid.pos.y ) >= 1. ) { 
    boid.vel.y *= -1.; 
  }
  if( abs( boid.pos.x ) > 1. ) {
    boid.vel.x *= -1.;
  }

  state[ idx ] = boid;
}
```

Last but not least, let's implement one of the extra rules that Reynolds describes. In this case
we'll limit the speed of our boids so that they don't go haywire. You can put this fragment right before the 
next position is calculated in the compute shader.

```wgsl
// limit speed
if( length( boid.vel ) > 5. ) {
  boid.vel = (boid.vel / length(boid.vel)) * 5.;
}
```

That's it! You should have a reasonable setup for boids at this point, now you can play around with the rendering,
the number of agents, and the various coefficients used in the algorithm. You might also try exploring some
of the other optional rules of boids described in the Reynolds article.
