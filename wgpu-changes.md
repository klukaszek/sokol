# Sokol WGPU Compute Shader Implementation Changes

## June 4th 2024

Added compute shader related fields to necessary WGPU Backend data structures.
- _sg_wgpu_pipeline_t
- _sg_wgpu_backend_t

Changed _sg_wgpu_create_pipeline()

Added "void sg_wgpu_begin_compute_pass(void)"

Added "void sg_wgpu_end_compute_pass(void)"

Added "void sg_wgpu_dispatch_compute_shader(uint32_t x, uint32_t y, uint32_t z)"

### Current issue

```
compute-wgpu.html:1 Initializing Nano Sokol app...
compute-wgpu.html:1 nano_font_index = 0
compute-wgpu.html:1 sapp_width() = 1920
compute-wgpu.html:1 sapp_height() = 741
compute-wgpu.html:1 The number of dynamic uniform buffers (12) exceeds the maximum per-pipeline-layout limit (8).
 - While validating binding counts
 - While validating [BindGroupLayoutDescriptor]
 - While calling [Device].CreateBindGroupLayout([BindGroupLayoutDescriptor]).

compute-wgpu.html:1 [Invalid BindGroupLayout (unlabeled)] is invalid.
 - While validating [BindGroupDescriptor] against [Invalid BindGroupLayout (unlabeled)]
 - While calling [Device].CreateBindGroup([BindGroupDescriptor]).

compute-wgpu.html:1 On entries[2]: binding index (48) was specified by a previous entry.
 - While validating [BindGroupLayoutDescriptor]
 - While calling [Device].CreateBindGroupLayout([BindGroupLayoutDescriptor]).

compute-wgpu.html:1 [Invalid BindGroupLayout (unlabeled)] is invalid.
 - While calling [Device].CreatePipelineLayout([PipelineLayoutDescriptor]).

compute-wgpu.html:1 [Invalid PipelineLayout (unlabeled)] is invalid.
 - While calling [Device].CreateRenderPipeline([RenderPipelineDescriptor "sokol-imgui-pipeline"]).

compute-wgpu.html:1 On entries[2]: binding index (48) was specified by a previous entry.
 - While validating [BindGroupLayoutDescriptor]
 - While calling [Device].CreateBindGroupLayout([BindGroupLayoutDescriptor]).

compute-wgpu.html:1 [Invalid BindGroupLayout (unlabeled)] is invalid.
 - While calling [Device].CreatePipelineLayout([PipelineLayoutDescriptor]).

compute-wgpu.html:1 [Invalid PipelineLayout (unlabeled)] is invalid.
 - While calling [Device].CreateRenderPipeline([RenderPipelineDescriptor "sokol-imgui-pipeline-unfilterable"]).

compute-wgpu.html:1 [Invalid BindGroupLayout (unlabeled)] is invalid.
 - While validating [BindGroupDescriptor] against [Invalid BindGroupLayout (unlabeled)]
 - While calling [Device].CreateBindGroup([BindGroupDescriptor]).

[Invalid BindGroup (unlabeled)] is invalid.
 - While encoding [RenderPassEncoder (unlabeled)].SetBindGroup(0, [BindGroup], 12, ...).

[Invalid BindGroup (unlabeled)] is invalid.
 - While encoding [RenderPassEncoder (unlabeled)].SetBindGroup(0, [BindGroup], 12, ...).

...

[Invalid BindGroup (unlabeled)] is invalid.
 - While encoding [RenderPassEncoder (unlabeled)].SetBindGroup(0, [BindGroup], 12, ...).

[Invalid CommandBuffer] is invalid.
 - While calling [Queue].Submit([[Invalid CommandBuffer]])

[Invalid CommandBuffer] is invalid.
 - While calling [Queue].Submit([[Invalid CommandBuffer]])

...

[Invalid CommandBuffer] is invalid.
 - While calling [Queue].Submit([[Invalid CommandBuffer]])

compute-wgpu.html:1 WebGPU: too many warnings, no more warnings will be reported to the console for this GPUDevice.
```

The problem here is that the BindGroupLayout is determining that any Layout needs to have 4 dynamic UBs per type of shader. This results in 12 dynamic UBs (4 VS, 4 FS, 4 CS). WGPU can only support 8 UBs per pipeline layout. I must come up with a solution to work around the Sokol WGPU implementation and define pipelines with VS and FS UBs, and CS and FS UBs. This gets us to have 8 UBs for the pipeline layout but I currently do not have any ideas as to how I'll approach this. Perhaps I can look into unions.

If we keep SG_NUM_SHADER_STAGES = 2, then the problem is solved, but then compute shaders won't actually work yet due to some logic reusing SG_NUM_SHADER_STAGES for other purposes that don't validate the # of dynamic UBs.

See _sg_shader_desc_defaults() to fix one of these SG_NUM_SHADER_STAGES related problems. The code is iterating from [0, SG_NUM_SHADER_STAGES) and assigns the defaults for the VS and FS stages. Normally this works because the macro = 2. When introducing a 3rd possibility, the logic changes a bit. I am considering just using the VS memory allocations to store all the data for the CS and just run the ComputePipeline and have it interact with the Pipeline that thinks it has a vertex shader (but is actually a compute shader).

_sg_shader_common_init()

_sg_wgpu_create_shader()

Check _SOKOL_PRIVATE void _sg_wgpu_uniform_buffer_init(const sg_desc *desc) to verify BindGroupLayout visibility. Here it seems the only options are VS and FS, but CS should become an option in the future.
