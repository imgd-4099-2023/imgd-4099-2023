# Physarum (Slime Mold)
Today we’ll be looking at slime molds. Which are awesome. Really! Single-celled organisms with thousands / millions of nuclei.

- [Slime Mold Smarts](https://www.scientificamerican.com/article/brainless-slime-molds/)
- [Are you smarter than a slime mold?](https://www.youtube.com/watch?v=K8HEDqoTPgk)
- [Coding Adventure: Ant and Slime Simulations (thanks Eli!)](https://www.youtube.com/watch?v=X-iSQQgOd1A&t=9s)
- [Twitter feed for Sage Jenson](https://twitter.com/mxsage)

As it turns out, our model for Langton’s Ants gets us… most of the way to being able to do Physarum simulations. The core concept in Langton’s algorithm—agents move through a space leave trails of chemicals which in turn influence how agents move—is  the majority of what the Physarum simulation is all about. There’s a couple of other elements (of course):

1. Physarum nuclei have *sensors* they can use to detect chemical trails at locations that are distant from them
2. The chemical trails will both *diffuse* and *decay* over time. This requires a separate shader pass… to keep things simple, we’ll use the Laplace transform from the reaction diffusion model for diffusion; gaussian blurs are also recommended.
3. We’re going to use millions (!!!) of vertices to represent the nuclei of a slime mold. 

Here are a few different resources to learn more about the algorithm we’ll create:

- https://cargocollective.com/sagejenson/physarum
- https://jbaker.graphics/writings/physarum.html
- [Original paper on the simulation](http://eprints.uwe.ac.uk/15260/1/artl.2010.16.2.pdf)

We'll start with the  [Langton’s Ants demo from seagulls](https://github.com/charlieroberts/seagulls/blob/main/demos/15_vants/main.js) 
and transform / extend it into slime mold.

The algorithm for the compute shader is as follows for each agent:

1. “Look” or “sense” ahead, to the left, and to the right, to see where the greatest concentration of chemicals is. If left and right values are equal, randomly pick one.
2. Rotate the agent’s heading towards the greatest concentration of chemicals
3. Move agent one pixel along the new heading

Pretty simple algorithm, but it takes about forty lines of code implement.
We’ll create a couple of utility functions in addition to our main program. 
The first will accept a 2D direction and rotate it a specified number of radians (not degrees!). 
The second will be used to “sense” chemical deposits at a given distance and direction away from the agent.

Our uniforms / buffers will be the same from our Langton simulation, but we'll add a `frame` uniform so
we can animate some parameters of the algorithm.

```js
import { default as seagulls } from '../../seagulls.js'

const NUM_AGENTS       = 2048*2048,
      GRID_SIZE        = 1,
      W                = Math.round( window.innerWidth  / GRID_SIZE ),
      H                = Math.round( window.innerHeight / GRID_SIZE ),
      WORKGROUP_SIZE   = 8,
      dc               = Math.ceil( Math.sqrt( NUM_AGENTS/64 ) ),
      DISPATCH_COUNT   = [ dc,dc, 1 ],
      DISPATCH_COUNT_2 = [ Math.ceil(W/8), Math.ceil(H/8), 1 ],
      LEFT = .0, RIGHT = 1.,
      FADE = .0125

const render_shader = seagulls.constants.vertex + `
struct VertexInput {
  @location(0) pos: vec2f,
  @builtin(instance_index) instance: u32,
}

@group(0) @binding(0) var<uniform> frame: f32;
@group(0) @binding(1) var<storage> vants: array<f32>;
@group(0) @binding(2) var<storage> pheromones: array<f32>;

@fragment 
fn fs( @builtin(position) pos : vec4f ) -> @location(0) vec4f {
  let grid_pos = floor( pos.xy / ${GRID_SIZE}.);
  
  let pidx = grid_pos.y  * ${W}. + grid_pos.x;
  let p = pheromones[ u32(pidx) ];

  return vec4f( vec3(p*80.),1. );
}`

const compute_shader =`
struct Vant {
  pos: vec2f,
  dir: f32,
  mode: f32
}

@group(0) @binding(0) var<uniform> frame: f32;
@group(0) @binding(1) var<storage, read_write> vants: array<Vant>;
@group(0) @binding(2) var<storage, read_write> pheromones: array<f32>;
@group(0) @binding(3) var<storage, read_write> render: array<f32>;

fn vantIndex( cell:vec3u ) -> u32 {
  let size = ${WORKGROUP_SIZE}u;
  return cell.x + (cell.y * size); 
}

fn pheromoneIndex( vant_pos: vec2f ) -> u32 {
  return u32( round(vant_pos.y * ${W}. + vant_pos.x) );
}

fn readSensor( pos:vec2f, dir:f32, angle:f32, distance:vec2f ) -> f32 {
  let read_dir = vec2f( sin( (dir+angle) * ${Math.PI*2} ), cos( (dir+angle) * ${Math.PI*2} ) );
  let offset = read_dir * distance;
  let index = pheromoneIndex( round(pos+offset) );
  return pheromones[ index ];
}

@compute
@workgroup_size(${WORKGROUP_SIZE}, ${WORKGROUP_SIZE},1)

fn cs(@builtin(global_invocation_id) cell:vec3u)  {
  let turn = .075;
  let index = vantIndex( cell );
  var vant:Vant = vants[ index ];

  var pIndex:u32 = pheromoneIndex( round(vant.pos) );

  let sensorDistance = vec2f( 7. + sin(frame/200.) * 3. ); 

  let left     = readSensor( vant.pos, vant.dir, -turn, sensorDistance );
  let forward  = readSensor( vant.pos, vant.dir, 0.,    sensorDistance );
  let right    = readSensor( vant.pos, vant.dir, turn,  sensorDistance );
  
  if( left > forward && left > right ) {
    vant.dir -= turn; 
  }else if( right > left && right > forward ) { 
    vant.dir += turn;
  }else if ( right == left ) { 
    let rand = fract( sin( vant.pos.x ) * 100000.0 );
    if( rand > .5 ) {
      vant.dir += turn; 
    }else{
      vant.dir -= turn;
    }
  }
  
  let advance_dir = vec2f( sin( vant.dir * ${Math.PI*2} ), cos( vant.dir * ${Math.PI*2} ) );
  vant.pos = vant.pos + advance_dir * .1 ; 
  pIndex = pheromoneIndex( round(vant.pos) );

  pheromones[ pIndex ] += .01;

  vants[ index ] = vant;
}`

const NUM_PROPERTIES = 4 // must be evenly divisble by 4!
const pheromones   = new Float32Array( W*H ) // hold pheromone data
const vants_render = new Float32Array( W*H ) // hold info to help draw vants
const vants        = new Float32Array( NUM_AGENTS * NUM_PROPERTIES ) // hold vant info

for( let i = 0; i < NUM_AGENTS * NUM_PROPERTIES; i+= NUM_PROPERTIES ) {
  vants[ i ]   = Math.floor( (LEFT+Math.random()*RIGHT) * W ) // x
  vants[ i+1 ] = Math.floor( (LEFT+Math.random()*RIGHT) * H ) // y
  vants[ i+2 ] = Math.floor( Math.random() * 4 ) / 4 // direction 
  vants[ i+3 ] = Math.random() 
}

const sg = await seagulls.init()

let frame = 0
sg.buffers({
    vants,
    pheromones,
    pheromonesB:pheromones,
    vants_render
  })
  .uniforms({ frame })
  .onframe( ()=> sg.uniforms.frame = frame++ )
  .backbuffer( false )
  .blend( true )
  .compute( compute_shader, DISPATCH_COUNT )
  .render( render_shader )
  .run(1)
```

Just running this will get you some of the behavior of physarum. However, an important component
is that the pheromones *diffuse and decay*. In order to do this, we need to run a second compute shader;
instead of operating on all the agents this shader will operate over our pheromone buffer. We'll use
the same diffusion kernel we used in our Reaction Diffusion assignment to diffuse the pheromones, and
then decay them by a constant amount. 

Add in the following to our `main.js` file:

```js
const compute2 = `
@group(0) @binding(0) var<uniform> frame: f32;
@group(0) @binding(1) var<storage, read_write> pheromones: array<f32>;
@group(0) @binding(2) var<storage, read_write> pheromones_write: array<f32>;

fn getP( x:u32,y:u32 ) -> f32 {
  let idx = u32( y * ${W}u + x );
  return pheromones[ idx ];
}

@compute
@workgroup_size(${WORKGROUP_SIZE}, ${WORKGROUP_SIZE},1)

fn cs(@builtin(global_invocation_id) cell:vec3u)  {
  let x = cell.x;
  let y = cell.y;

  var state:f32 = getP( x,y ) * -1.;
  state += getP(x - 1u, y) * 0.2;
  state += getP(x - 1u, y - 1u) * 0.05;
  state += getP(x, y - 1u) * 0.2;
  state += getP(x + 1u, y - 1u ) * 0.05;
  state += getP(x + 1u, y) * 0.2;
  state += getP(x + 1u, y + 1u ) * 0.05;
  state += getP(x, y + 1u ) * 0.2;
  state += getP(x - 1u, y + 1u ) * 0.05;

  let pIndex = y * ${W}u + x;
  pheromones[ pIndex ] = state * .4;
}
`
```

Hopefully that code looks familiar to you! Now that the shader has been defined, we can add a second call to
`sg.compute` as follows:

```js
  .compute( compute2, DISPATCH_COUNT_2, { buffers:['pheromones', 'pheromonesB'] })
```

... immediatley after the first call. You'll notice we've already setup a second `DISPATCH_COUNT_2` variable
at the top of our JS file to control how many times this compute shader is called based on the width and
height of our screen and grid size.

Now you can play with the various aspects of the algorithm... sensing distance, turn radius, speed of travel etc.

