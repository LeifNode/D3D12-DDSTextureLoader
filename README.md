# D3D12-DDSTextureLoader

## No longer works due to updates to the DirectX 12 API that prevent this method of texture creation. 
## There is now official support for DirectX12 in [DirectXTex's DDSTextureLoader12](https://github.com/Microsoft/DirectXTex/tree/master/DDSTextureLoader)

Light-weight runtime DDS file loader for Direct3D 12.

Port of the Microsoft DDSTextureLoader to support the Direct3D 12 runtime. The original version for Direct3D 11 can be found at https://github.com/Microsoft/DirectXTex

### Functionality

The functionality is the same as the original DDSTextureLoader except that it does not currently support automatic mip-mapping so mipmaps will need to be generated in an external tool.

* Supports loading DDS textures with mip-maps
* Supports most color formats
* Supports compressed formats
* Supports 2D textures and cube maps (1D textures, 3D textures, and array types should also be supported, but have not been tested)

Automatic mipmap generation is not supported at the moment because Direct3D 12 does not have an equivalent to GenerateMips since the same behavior could be handled through compute shaders. 

### Performance Note

Resources are created on committed heaps with the type **D3D12_HEAP_TYPE_UPLOAD**. Committed heaps are the most similar to D3D11 resources in that the heap is bound to the lifetime of the resource. Upload heaps are intended for uploading data to the GPU, and are not intended to be used for frequent shader access. Resources that are frequently sampled from should be allocated in heaps with the type **D3D12_HEAP_TYPE_DEFAULT**.

While it is possible to sample textures created directly from D3D12-DDSTextureLoader, there can be severe performance reductions depending on the hardware and driver. In some tests it took about 30x as long to sample from textures in upload heaps as compared to default heaps on my system.

It is currently left to the user to initialize a default heap and copy the texture from the upload heap.

### Usage

The functions to load textures are similar to the original DDSTextureLoader, but with some changes to handle how D3D12 handles resource views.

*CreateDDSTextureFromFile* is used to load DDS textures from a specified file path:

```c++
HRESULT CreateDDSTextureFromFile(ID3D12Device* d3dDevice,
		const wchar_t* szFileName,
		ID3D12Resource** texture,
		D3D12_SHADER_RESOURCE_VIEW_DESC* textureView,
		size_t maxsize = 0,
		DDS_ALPHA_MODE* alphaMode = nullptr
		);
```

*CreateDDSTextureFromMemory* is used to load textures from a block of memory:

```c++
HRESULT CreateDDSTextureFromMemory(ID3D12Device* d3dDevice,
		const uint8_t* ddsData,
		size_t ddsDataSize,
		ID3D12Resource** texture,
		D3D12_SHADER_RESOURCE_VIEW_DESC* textureView,
		size_t maxsize = 0,
		DDS_ALPHA_MODE* alphaMode = nullptr
		);
```

**texture:** Pointer to the pointer that will point to the committed resource

**textureView:** Pointer to the structure that describes the resource view. This is the primary difference between the DDSTextureLoader for D3D11. D3D12 has changed the resource view model such that the shader resource view objects that needed to be constructed in D3D11 are no longer necessary since the resource lifetime is managed through its heap. [**D3D12_SHADER_RESOURCE_VIEW_DESC**](https://msdn.microsoft.com/en-us/library/windows/desktop/dn770406%28v=vs.85%29.aspx) structures describe the type of view to shaders. This parameter is optional.

### Loading and Transfer from Upload Heap to Default Heap

```c++
ComPtr<ID3D12Device> Device;
ComPtr<ID3D12Resource> exampleTextureUpload;
ComPtr<ID3D12Resource> exampleTexture;
//Command list used to initialize the state and resources
//Must be finished executing before the exampleTexture resource can be bound.
ComPtr<ID3D12GraphicsCommandList> InitCommandList; 
D3D12_SHADER_RESOURCE_VIEW_DESC exampleTextureViewDesc = {};

...

HRESULT hr = DirectX::CreateDDSTextureFromFile(
    Device.Get(), 
    L"exampleTex.dds", 
    exampleTextureUpload.ReleaseAndGetAddressOf(), 
    &exampleTextureViewDesc);

D3D12_RESOURCE_DESC exampleResourceDesc = exampleTextureUpload->GetDesc();

Device->CreateCommittedResource(
    &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),
    D3D12_HEAP_FLAG_NONE,
    &exampleResourceDesc,
    D3D12_RESOURCE_STATE_COMMON,
    nullptr,
    IID_PPV_ARGS(exampleTexture.GetAddressOf()));

D3D12_RESOURCE_BARRIER barrierDesc = {};

barrierDesc.Type = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
barrierDesc.Transition.pResource = exampleTexture.Get();
barrierDesc.Transition.Subresource = D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES;
barrierDesc.Transition.StateBefore = D3D12_RESOURCE_STATE_COMMON;
barrierDesc.Transition.StateAfter = D3D12_RESOURCE_STATE_COPY_DEST;

InitCommandList->ResourceBarrier(1, &barrierDesc);
InitCommandList->CopyResource(exampleTexture.Get(), exampleTextureUpload.Get());

barrierDesc.Transition.StateBefore = D3D12_RESOURCE_STATE_COPY_DEST;
barrierDesc.Transition.StateAfter = D3D12_RESOURCE_STATE_GENERIC_READ;
//The texture must be in D3D12_RESOURCE_STATE_GENERIC_READ before it can be sampled from
InitCommandList->ResourceBarrier(1, &barrierDesc); 

//Tell the GPU that it can free the heap 
CommandList->DiscardResource(exampleTextureUpload.Get(), nullptr);

...

WaitForGPU();

//After the InitCommandList has finished executing it is safe to release the upload heap
exampleTextureUpload.Reset();

```

### Direct3D 12 Texture Examples

[HelloD3D12](https://github.com/shobomaru/HelloD3D12) has some samples for binding textures using the root signature in D3D12. 
