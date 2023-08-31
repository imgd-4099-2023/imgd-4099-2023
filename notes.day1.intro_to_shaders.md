# Diving into some basic shader programming

[wgsl_live](https://charlieroberts.github.io/wgsl_live/) is an environment for live coding WGSL shaders. We'll use it to do some initial WGSL experiments creating waveforms.

Our first shader in WGSL:
```c
@fragment 
fn fs( @builtin(position) pos : vec4f ) -> @location(0) vec4f {
  // whatever we return from our main fragment shader
  // function will yield the color for each pixel
  return vec4( 1., 0., 0., 1. );
}
```

It is important to note that most numbers are floating point and **must have a decimal point to indicate this**. 
It is really easy to forget this, especially coming from JavaScript, so be careful! In the above example, the `pos` variable
represents the position of our fragment on the screen, *in pixels*. We'll convert this to normalized coordinates later.

## Creating an oscillator
wgsl_live comes pre-programmed with special **uniforms** (a uniform is a value transferred from the CPU to the GPU). One such value is `frame`. We can use this to make a simple sine wave:

```c
@fragment 
fn fs( @builtin(position) pos : vec4f ) -> @location(0) vec4f {
  let frequency = 4.;
  let color = sin( frame / 60. * frequency );
  return vec4f( color,color,color, 1. );
}
```

However, you’ll notice that this spends a half of each cycle as black, because the `sin()` function returns values between -1 – 1. We can provide a bias and a scalar to get this to a range of 0–1:

```c
@fragment 
fn fs( @builtin(position) pos : vec4f ) -> @location(0) vec4f {
  // create normalized position coordinates in range 0-1
  let frequency = 4.;
  let bias = .5;
  let gain = .5;
  let color  = bias + sin( frame / 60. * frequency ) * gain;
  return vec4f( color,color,color, 1. );
}
```

## Varying our oscillator over space
As we mentioned earlier, the `pos` variable reports the current fragment position in pixels. wgsl_live provides the uniform `res` to tell you the resolution of your drawing surface.
We can use these together to get a normalized (0--1) coordinate for our fragement, and can use this to make a classic video oscillator on our screen.

```c
@fragment 
fn fs( @builtin(position) pos : vec4f ) -> @location(0) vec4f {
  // create normalized position coordinates in range 0-1
  let p = pos.xy / res;
  let frequency = 30.;
  let bias = .5;
  let gain = .5;
  let color  = bias + sin( p.x * frequency + (frame/60.) ) * gain;
  return vec4f( color,0.,0., 1. );
}
```

… and now’s where things start to get fun. We can easily use the x and y values to create all sorts of interesting variations on this. Try this one:

```c
@fragment 
fn fs( @builtin(position) pos : vec4f ) -> @location(0) vec4f {
  // create normalized position coordinates in range 0-1
  let p = pos.xy / res;
  let frequency = 10.;
  let color  = sin( p.x / p.y * frequency + (frame/60.) );
  return vec4f( color,0.,0., 1. );
}
```

What happens when you multiply `p.x` by `p.y` instead of dividing it? What happens when you multiply the entire output of the `sin()` function by `p.y`? All sort of fun variations to play with.

## Classic live shader programming videos:

- Hexler - [codelife - glsl live-coding editor test #5 on Vimeo](https://vimeo.com/51993089)
- Shawn Lawson - [RISC Chip - Mint: Kindohm. Visuals: Obi Wan Codenobi on Vimeo](https://vimeo.com/192920872)

## Let’s make a Hexler-style sine wave and use an FFT
```c
@fragment 
fn fs( @builtin(position) pos : vec4f ) -> @location(0) vec4f {
  // create normalized position coordinates in range 0-1
  var p : vec2f = pos.xy / res;
  let frequency = 2.;
  let thickness = .05;
  p.x += sin( p.y * frequency + (frame/60.) * 4. ) * .5 - .5; 
  let color  = abs( thickness / p.x );
  return vec4f( color,0.,0., 1. );
}
```

In this case, we have to use the `var` keyword for `p` and declare its type, because it's not a constant variable. This is one big difference between
WGSL and GLSL. OK, that’s fun, but the only thing better than a sine wave is twenty sine waves:

```c
@fragment 
fn fs( @builtin(position) pos : vec4f ) -> @location(0) vec4f {
  // create normalized position coordinates in range 0-1
  var p : vec2f = -1. + (pos.xy / res) * 2.;
  var color: vec4f = vec4f(0.);
  let frequency = 2.;
  let gain = 1.;
  let thickness = .025;
  for( var i:f32 = 0.; i < 20.; i+= 1. ) {
    p.x += sin( p.y + (frame/60.) * i ) * gain; 
    color += abs( thickness / p.x );
  }
  
  return color;
}
```

### Adding the FFT
In wgsl_live, we can get access to the data from our laptop microphones by clicking on the audio button at the bottom of the screen. Once you’ve granted your browser access to use the microphone, the FFT data will then be available in a `vec3f` uniform named `audio`, where `audio[0]` is the low-frequency content and `audio[2]` is the high frequency content. 
Try setting the value of `gain` in the above script to use one of these bands and have fun watching the results… and then try messing around with the formula in other ways, 
dividing instead of of multiplying, use fewer or more iterations of the for loop etc.

*IMPORTANT: The audio input function does not appear to be working in Firefox under Linux. Try running the Force in a different browser if you have trouble getting `bands` to work correctly.*
