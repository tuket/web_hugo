---
layout: post
title: Create a simple BRDF plotter with Untity
date: 2017-05-13
---

I needed a tool to plot analytical BRDFs.
~~I was surprised I wasn't able to find anything that could fit my requirements~~(shame on me: check out [the one from Disney](https://github.com/wdas/brdf)).
So I created my own interactive BRDF plotter using Unity.
It's pretty easy to create your own BRDF plotter (if you don't have big expectations on it).
The goal is to make something useful in a couple hours.


## Displacing vertices of a sphere

What we are going to do is super easy. Create a sphere, evaluate the BRDF, displace the vertices by that amount. We can do this in either the vertex, or geometry shader.

Unfortunately spheres in Unity have fixed resolution and it is quite low.
Since we don't want our BRDF to look spiky, we will import a higher resolution sphere.
We can make a sphere in an external modeling tool like blender and import it. Feel free to take this [sphere](https://drive.google.com/file/d/0BwBC4DUnZLBXbU9xNWdNUWs4bnM/view?usp=sharing).

Also we can increse the resolution of the sphere using geometry shaders. But if we want to make something quick we can skip this part.

The first thing we are going to do is create a new material and a new shader (just right click in the folder you want them to be created and select create>). The type of shader we are going to use is unlit.

![image](http://i.imgur.com/rs5Ooi0.png)

Then open the shader source. You should see a function called `vert`, which is actually the vertex shader. If you placed the sphere at the origin of coordinates you can just evaluate the brdf, then set the position of that vertex at a distance proportional to the BRDF value.
You could also set the color of the vertex acording to the BRDF value.

Here is an example of a vertex shader.

```c#
v2f vert (appdata v)
{
	v2f o;

	float3 L = normalize(v.vertex.xyz);
	if(dot(L, float3(0, 1, 0)) > 0)
	{
		// set distance proportional to the BRDF value
		float3 V = normalize(_Ray.xyz);
		float intensity = someBRDF(V, L);
		v.vertex = float4(intensity * L, 1);
		
		// put some color to the vertex
		o.color = float4(0, intensity, 0, 1);
	}
	else
	{
		// this part of the BRDF shouldn't be plotted
		// but some people might find utility in
		float3 V = normalize(_Ray.xyz);
		float intensity = someBRDF(V, L);
		v.vertex = float4(intensity * L, 1);
		o.color = float4(1, 0, 0, 1);
	}

	o.vertex = UnityObjectToClipPos(v.vertex);
	o.uv = TRANSFORM_TEX(v.uv, _MainTex);
	UNITY_TRANSFER_FOG(o, o.vertex);

	return o;
}
```

At the top of the code you can add `Properties` to the shader.
You could use these to add some parameters of the BRDF (e.g. roughness) and be able to modify them in real-time.

```c#
Properties
{
    _MainTex ("Texture", 2D) = "white" {}
    _Alpha ("Alpha", Float) = 0.5
    _Ray("Ray", Vector) = (0, 1, 0, 0)
    _R0 ("R0", Float) = 0.5
}
```

In the previous example I have added `_Alpha` for the roughness, `_Ray` for the incident ray  direction, and `_R0` for the base refectance at zero degrees. You can add as many as you want and modify them in real-time, in the editor, or through script.

## Additional improvements

We could make a lot of improvements to our BRDF plotter. I have implemented only some basic features. Here are some ideas:
* Orbital camera.
* Zoom
* Controls to change the direction of incident ray.
* Change the type of BRDF in real-time.
* Measurement tools.
* Statistics and other info.
* Use geometry shaders for more resolution.
* Whatever you want.

![BRDF plot](http://i.imgur.com/sJq40wS.gif)

Once you have implemented the basic structure you can just replace the BRDF function for another.

Basic plotter example project: [download](https://drive.google.com/open?id=0BwBC4DUnZLBXU0NFMlZSU2hSQWs).


