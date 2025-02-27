<!--
- Copyright (c) 2022, The Khronos Group
-
- SPDX-License-Identifier: Apache-2.0
-
- Licensed under the Apache License, Version 2.0 the "License";
- you may not use this file except in compliance with the License.
- You may obtain a copy of the License at
-
-     http://www.apache.org/licenses/LICENSE-2.0
-
- Unless required by applicable law or agreed to in writing, software
- distributed under the License is distributed on an "AS IS" BASIS,
- WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
- See the License for the specific language governing permissions and
- limitations under the License.
-
-->
### HPP Separate image sampler<br/>
A transcoded version of the API sample [Separate image sampler](https://github.com/KhronosGroup/Vulkan-Samples/tree/master/samples/api/separate_image_sampler) that illustrates the usage of the C++ bindings of vulkan provided by vulkan.hpp.

# Separating samplers and images

This is the tutorial as written in [Separate image sampler](https://github.com/KhronosGroup/Vulkan-Samples/tree/master/samples/api/separate_image_sampler), with code transcoded to functions and classes from vulkan.hpp.

This tutorial, along with the accompanying example code, shows how to separate samplers and images in a Vulkan application. Opposite to combined image and samplers, this allows the application to freely mix an arbitrary set of samplers and images in the shader.

In the sample code, a single image and multiple samplers with different options will be created. The sampler to be used for sampling the image can then be selected at runtime. As image and sampler objects are separated, this only requires selecting a different descriptor at runtime.

## In the application

From the application's point of view, images and samplers are always created separately. Access to the image is done via the image's `vk::ImageView`. Samplers are created using a `vk::Sampler` object, specifying how an image will be sampled.

The difference between separating and combining them starts at the descriptor level, which defines how the shader accesses the samplers and images.

A separate setup uses a descriptor of type `vk::DescriptorType::eSampledImage` for the sampled image, and a `vk::DescriptorType::eSampler` for the sampler, separating the image and sampler object:

```cpp
// Image info only references the image
vk::DescriptorImageInfo image_info({}, texture.image->get_vk_image_view().get_handle(), vk::ImageLayout::eShaderReadOnlyOptimal);

// Sampled image descriptor
vk::WriteDescriptorSet image_write_descriptor_set(base_descriptor_set, 1, 0, vk::DescriptorType::eSampledImage, image_info);

// One set for the sampled image
std::array<vk::WriteDescriptorSet, 2> write_descriptor_sets = {{
	{base_descriptor_set, 0, 0, vk::DescriptorType::eUniformBuffer, {}, buffer_descriptor},        // Binding 0 : Vertex shader uniform buffer
	image_write_descriptor_set                                                                     // Binding 1 : Fragment shader sampled image
}};
get_device()->get_handle().updateDescriptorSets(write_descriptor_sets, {});
```

For this sample, we then create two samplers with different filtering options:

```cpp
// Sets for each of the sampler
descriptor_set_alloc_info.pSetLayouts = &sampler_descriptor_set_layout;
for (size_t i = 0; i < sampler_descriptor_sets.size(); i++)
{
	sampler_descriptor_sets[i] = get_device()->get_handle().allocateDescriptorSets(descriptor_set_alloc_info).front();

	// Descriptor info only references the sampler
	vk::DescriptorImageInfo sampler_info(samplers[i]);

	vk::WriteDescriptorSet sampler_write_descriptor_set(sampler_descriptor_sets[i], 0, 0, vk::DescriptorType::eSampler, sampler_info);

	get_device()->get_handle().updateDescriptorSets(sampler_write_descriptor_set, {});
}
```

At draw-time, the descriptor containing the sampled image is bound to set 0 and the descriptor for the currently selected sampler is bound to set 1:

```cpp
// Bind the uniform buffer and sampled image to set 0
draw_cmd_buffers[i].bindDescriptorSets(vk::PipelineBindPoint::eGraphics, pipeline_layout, 0, base_descriptor_set, {});
// Bind the selected sampler to set 1
draw_cmd_buffers[i].bindDescriptorSets(vk::PipelineBindPoint::eGraphics, pipeline_layout, 1, sampler_descriptor_sets[selected_sampler], {});
...
draw_cmd_buffers[i].drawIndexed(index_count, 1, 0, 0, 0);
``` 

## In the shader

There are no changes in the shader code to get it working with vulkan.hpp.
With the above setup, the shader interface for the fragment shader also separates the sampler and image as two distinct uniforms:

```glsl
layout (set = 0, binding = 1) uniform texture2D _texture;
layout (set = 1, binding = 0) uniform sampler _sampler;
```

To sample from the image referenced by `_texture`, with the currently set sampler in '_sampler', we create a sampled image in the fragment shader at runtime using the `sampler2D` function.

```glsl
void main() 
{
    vec4 color = texture(sampler2D(_texture, _sampler), inUV);
}
```

## Comparison with combined image samplers

For reference, a combined image and sampler setup would differ for both the application and the shader. The app would use a single descriptor of type `VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER`, and set both image and sampler related values in the descriptor:

```cpp
// Descriptor info references image and sampler
vk::DescriptorImageInfo image_info(texture.sampler, texture.view, texture.image_layout);

vk::WriteDescriptorSet image_write_descriptor_set(descriptor_set, 1, {}, vk::DescriptorType::eCombinedImageSampler, image_info);
```

The shader interface only uses one uniform for accessing the combined image and sampler and also doesn't construct a `sampler2D` at runtime:

```glsl
layout (binding = 1) uniform sampler2D _combined_image;

void main() 
{
    vec4 color = texture(_combined_image, inUV);
}
```

Compared to the separated setup, changing a sampler in this setup would either require creating multiple descriptors with each image/sampler combination or rebuilding the descriptor.