# The Complete Three.js WebGPU Roadmap: From Beginner to Advanced

*A structured learning path for modern 3D web graphics development*

---

The web graphics landscape is evolving rapidly. With WebGPU emerging as the successor to WebGL and Three.js embracing this new API, now is the perfect time to level up your 3D development skills. This roadmap will guide you through the journey from absolute beginner to advanced WebGPU developer.

## Why WebGPU? Why Now?

Before diving into the roadmap, let's understand why this matters:

- **Performance**: WebGPU provides lower-level GPU access, enabling better performance for complex scenes
- **Compute Shaders**: Native support for GPU compute opens doors to physics simulations, particle systems, and AI inference
- **Modern Architecture**: Built on Vulkan, Metal, and D3D12—the same APIs powering AAA games
- **Future-Proof**: Major browsers are rolling out support, making this the standard for web graphics

## The Learning Path

### Phase 1: Foundations (Weeks 1-2)

#### JavaScript Essentials
Before touching Three.js, ensure you're comfortable with:
- ES6+ syntax (classes, modules, arrow functions)
- Async/await and Promises
- Basic DOM manipulation

#### 3D Math Fundamentals
You don't need a math degree, but understanding these concepts is crucial:
- **Vectors**: Position, direction, and velocity in 3D space
- **Matrices**: Transformations (translation, rotation, scale)
- **Coordinate Systems**: Local vs. world space

**Resources to Start**:
- Three.js official documentation
- 3Blue1Brown's "Essence of Linear Algebra" on YouTube
- Interactive tutorials at threejs-journey.com

---

### Phase 2: Three.js Basics (Weeks 3-5)

#### Core Concepts to Master

```javascript
// The essential Three.js setup
import * as THREE from 'three';

// 1. Scene - Container for all objects
const scene = new THREE.Scene();

// 2. Camera - Your viewpoint
const camera = new THREE.PerspectiveCamera(75, width / height, 0.1, 1000);

// 3. Renderer - Draws everything (use WebGPU!)
const renderer = new THREE.WebGPURenderer({ antialias: true });

// 4. Geometry + Material = Mesh
const geometry = new THREE.BoxGeometry(1, 1, 1);
const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 });
const cube = new THREE.Mesh(geometry, material);
scene.add(cube);

// 5. Animation Loop
function animate() {
  cube.rotation.x += 0.01;
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

#### Key Topics
- **Geometries**: Built-in shapes (Box, Sphere, Plane, Torus) and custom BufferGeometry
- **Materials**: MeshBasicMaterial → MeshStandardMaterial → MeshPhysicalMaterial progression
- **Lighting**: Ambient, Directional, Point, and Spot lights
- **Cameras**: Perspective vs. Orthographic, OrbitControls
- **Textures**: Loading images, UV mapping, texture properties

#### Practice Project
Build a simple 3D scene viewer with:
- Multiple geometric objects
- Different materials and textures
- Interactive camera controls
- Basic lighting setup

---

### Phase 3: WebGPU Transition (Weeks 6-7)

#### Setting Up WebGPU Renderer

```javascript
import * as THREE from 'three';
import { WebGPURenderer } from 'three/webgpu';

const renderer = new WebGPURenderer({ antialias: true });
await renderer.init(); // WebGPU requires async initialization

renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
document.body.appendChild(renderer.domElement);
```

#### Understanding the Differences
| Aspect | WebGL | WebGPU |
|--------|-------|--------|
| Shader Language | GLSL | WGSL/TSL |
| Compute Support | Limited (via hacks) | Native |
| Initialization | Synchronous | Asynchronous |
| Performance | Good | Better |
| Browser Support | Universal | Growing |

---

### Phase 4: TSL - Three.js Shading Language (Weeks 8-10)

This is where the magic happens. TSL lets you write shaders in JavaScript instead of GLSL strings.

#### The TSL Philosophy

```javascript
// Instead of GLSL strings like this:
const fragmentShader = `
  varying vec2 vUv;
  void main() {
    gl_FragColor = vec4(vUv, 0.5, 1.0);
  }
`;

// TSL uses JavaScript:
import { uv, vec4 } from 'three/tsl';

const colorNode = vec4(uv(), 0.5, 1.0);
material.colorNode = colorNode;
```

#### Core TSL Concepts

**1. Types and Conversions**
```javascript
import { float, vec2, vec3, vec4, color } from 'three/tsl';

const scalar = float(1.0);
const position = vec3(1, 2, 3);
const rgba = vec4(1, 0, 0, 1);
```

**2. Operations with Method Chaining**
```javascript
import { time, sin, uv } from 'three/tsl';

// Animated gradient
const animated = time.mul(2.0).sin().mul(0.5).add(0.5);
const gradient = uv().x.mul(animated);
```

**3. Uniforms for Dynamic Values**
```javascript
import { uniform } from 'three/tsl';

const progress = uniform(0);

// Update in animation loop
function animate() {
  progress.value = (performance.now() / 1000) % 1;
}
```

#### Practice Project
Create an animated custom material with:
- UV-based color gradients
- Time-based animations
- Fresnel rim lighting effect

---

### Phase 5: Advanced Materials (Weeks 11-13)

#### Node Material System

```javascript
import { MeshStandardNodeMaterial } from 'three/webgpu';
import {
  uv, color, normalMap, texture,
  mix, fresnel, float
} from 'three/tsl';

const material = new MeshStandardNodeMaterial();

// PBR properties as nodes
material.colorNode = texture(diffuseMap, uv());
material.roughnessNode = texture(roughnessMap, uv()).r;
material.metalnessNode = float(0.9);
material.normalNode = normalMap(texture(normalMapTex, uv()));

// Add custom effects
const rim = fresnel({ power: 2.0 });
material.emissiveNode = color(0x00ffff).mul(rim);
```

#### Material Types to Explore
- **MeshBasicNodeMaterial**: Unlit, simple
- **MeshStandardNodeMaterial**: PBR with roughness/metalness
- **MeshPhysicalNodeMaterial**: Advanced PBR (clearcoat, transmission, iridescence)
- **SpriteNodeMaterial**: For billboards and particles
- **LineNodeMaterial**: Custom line rendering

---

### Phase 6: Compute Shaders (Weeks 14-16)

This is WebGPU's superpower—running general-purpose code on the GPU.

#### GPU Particle System Example

```javascript
import {
  compute, storage, instanceIndex,
  vec3, float, If
} from 'three/tsl';

// Create storage buffers
const positionBuffer = storage(positionArray, 'vec3', count);
const velocityBuffer = storage(velocityArray, 'vec3', count);

// Define compute shader
const computeParticles = Fn(() => {
  const position = positionBuffer.element(instanceIndex);
  const velocity = velocityBuffer.element(instanceIndex);

  // Apply gravity
  velocity.y.subAssign(0.01);

  // Update position
  position.addAssign(velocity);

  // Ground collision
  If(position.y.lessThan(0), () => {
    position.y.assign(0);
    velocity.y.mulAssign(-0.8); // Bounce
  });
})().compute(count);

// Run in animation loop
await renderer.computeAsync(computeParticles);
```

#### Use Cases
- **Particle Systems**: Thousands of particles with physics
- **Procedural Generation**: Terrain, textures, meshes
- **Physics Simulation**: Cloth, fluids, rigid bodies
- **Data Processing**: Image filters, scientific visualization

---

### Phase 7: Post-Processing (Weeks 17-18)

#### Building an Effects Pipeline

```javascript
import {
  pass, bloom, gaussianBlur,
  fxaa, vignetteNode
} from 'three/tsl';
import { PostProcessing } from 'three/webgpu';

const postProcessing = new PostProcessing(renderer);
const scenePass = pass(scene, camera);

// Chain effects
let result = scenePass;
result = bloom(result, 0.5, 0.4); // strength, threshold
result = vignetteNode(result, 0.5); // intensity
result = fxaa(result);

postProcessing.outputNode = result;

// Render
function animate() {
  postProcessing.render();
}
```

#### Essential Effects
- **Bloom**: Glowing highlights
- **Depth of Field**: Camera focus simulation
- **Color Grading**: Cinematic looks
- **Anti-aliasing**: FXAA, SMAA
- **Motion Blur**: Smooth fast movement

---

### Phase 8: Real-World Projects (Weeks 19+)

Now combine everything into portfolio-worthy projects:

#### Project Ideas

**1. Interactive Product Viewer**
- PBR materials with environment mapping
- Smooth camera animations
- Configurable options (colors, parts)
- Performance optimized for mobile

**2. Procedural Planet Generator**
- Compute shaders for terrain generation
- Atmospheric scattering
- Day/night cycle with city lights
- LOD (Level of Detail) system

**3. Particle Art Installation**
- 100k+ GPU particles
- Audio reactivity
- Custom post-processing
- Exportable as video

**4. Real-time Data Visualization**
- 3D charts and graphs
- Animated transitions
- Interactive exploration
- WebSocket data updates

---

## Tips for Success

### 1. Start Simple, Iterate
Don't jump straight to complex shaders. Master each phase before moving on.

### 2. Read the Source
Three.js examples are goldmines. Study how built-in effects are implemented.

### 3. Debug Strategically
- Use `console.log(node.build())` to see generated shader code
- Chrome DevTools has WebGPU profiling
- Start with simple values, add complexity gradually

### 4. Performance Mindset
- Profile early and often
- Batch draw calls where possible
- Use instancing for repeated objects
- Leverage compute shaders for heavy math

### 5. Stay Updated
Three.js WebGPU support is actively developed. Follow:
- Three.js GitHub releases
- @maboroshin_'s Twitter for TSL updates
- The Three.js Discord community

---

## Essential Resources

### Documentation
- [Three.js Documentation](https://threejs.org/docs/)
- [WebGPU Fundamentals](https://webgpufundamentals.org/)
- [TSL Examples](https://threejs.org/examples/?q=webgpu)

### Learning Platforms
- Three.js Journey (Bruno Simon)
- Discover Three.js (Lewy Blue)
- Shader School on GitHub

### Community
- Three.js Discord
- r/threejs on Reddit
- Stack Overflow [three.js] tag

---

## Conclusion

The path from Three.js beginner to WebGPU expert is challenging but rewarding. WebGPU represents the future of web graphics, and Three.js makes it accessible through TSL's intuitive JavaScript-based shading.

Remember: every expert was once a beginner. Start with the fundamentals, build projects that excite you, and don't be afraid to experiment. The web 3D community is welcoming, and there's never been a better time to join.

**Ready to start?** Clone the [WebGPU Three.js TSL Skill](https://github.com/dgreenheck/webgpu-threejs-tsl-skill) for comprehensive documentation, working examples, and templates to accelerate your journey.

Happy coding! 🚀

---

*Written for developers embarking on their Three.js WebGPU journey. Last updated: January 2025.*
