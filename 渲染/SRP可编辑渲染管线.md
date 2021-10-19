# SRP(Scriptable Render Pipeline)
SRP允许你编写C#代码来控制Unity的每帧渲染。让Unity在渲染的时候看起来不那么黑盒，与最初的内置渲染管线不同，SRP允许查看和精确的控制在渲染过程中发生了什么。
Unity提供了两种预制的自定义可编辑渲染管线：
  - URP 从移动端到PCs
  - HDRP 使用基于物理的光照技术来提供高保真的图像，Compute Shader兼容能力
SRPAPI为许多熟悉的统一结构提供了一个新的接口，包括：
  - Lights
  - Materials
  - Cameras
  - Command Buffer
Camera Components 提供了2个摄像机组件
  - Free Camera
  - Camera Switcher
Look Dev 只在编辑器下有用 而且必须使用HDRP 的HDRI材质

## 基本流程实现(可以正常渲染物体)
1. 实现MyPipeline 继承自RenderPipeline 在这里实现具体的渲染流程，视椎剪裁->绘制配置(绘制顺序)->过滤设置(设置透明或者非透明物体,设置层级，寻找Tag=LightMode)
  ~~~
  using System.Collections;
  using System.Collections.Generic;
  using UnityEngine;
  using UnityEngine.Rendering;
  public class MyPipeline : RenderPipeline
  {

      protected override void Render(ScriptableRenderContext context, Camera[] cameras)
      {
          foreach (var camera in cameras)
          {
              //设置渲染相关相机参数,包含相机的各个矩阵和剪裁平面等
              context.SetupCameraProperties(camera);

              //剪裁，这边应该是相机的视锥剪裁相关。
              //自定义一个剪裁参数，cullParam类里有很多可以设置的东西。我们先简单采用相机的默认剪裁参数。
              ScriptableCullingParameters cullParam = new ScriptableCullingParameters();
              //直接使用相机默认剪裁参数
              camera.TryGetCullingParameters(out cullParam);
              //非正交相机
              cullParam.isOrthographic = false;
              //获取剪裁之后的全部结果(其中不仅有渲染物体，还有相关的其他渲染要素)
              CullingResults cullResults = context.Cull(ref cullParam);

              //渲染时，会牵扯到渲染排序，所以先要进行一个相机的排序设置，这里Unity内置了一些默认的排序可以调用
              SortingSettings sortSet = new SortingSettings(camera) {criteria = SortingCriteria.CommonOpaque};
              //这边进行渲染的相关设置，需要指定渲染的shader的光照模式(就是这里，如果shader中没有标注LightMode的
              //话，使用该shader的物体就没法进行渲染了)和上面的排序设置两个参数
              DrawingSettings drawSet = new DrawingSettings(new ShaderTagId("Always"), sortSet);

              //这边是指定渲染的种类(对应shader中的Rendertype)和相关Layer的设置(-1表示全部layer)
              FilteringSettings filtSet = new FilteringSettings(RenderQueueRange.opaque, -1);
              //裁剪结果、绘制设置、过滤设置
              context.DrawRenderers(cullResults, ref drawSet, ref filtSet);
              //绘制天空球
              context.DrawSkybox(camera);
              //开始执行上下文
              context.Submit();
          }
      }
  }
  ~~~

2. 实现MyPipelineAssets 继承自RenderPipelineAsset

  ~~~
  using UnityEngine;
  using UnityEngine.Rendering;

  [CreateAssetMenu(menuName = "Rendering/MyPipeline")]
  public class MyPipelineAsset : RenderPipelineAsset
  {
      protected override RenderPipeline CreatePipeline()
      {
          //返回上一个脚本自定义的管线
          return new MyPipeline();
      }
  }

  ~~~

3. 实现不带光的Shader 在SRP之后会使用HLSL语言(这个不带光的Shader)
  ~~~
  Shader "SRPStudy/UnlitTexture"
  {
  	Properties
  	{
  		_Color("Color Tint", Color) = (0.5,0.5,0.5)
  		_MainTex("MainTex",2D) = "white"{}
  	}

  	HLSLINCLUDE
  	#include "UnityCG.cginc"

  	uniform float4 _Color;
  	sampler2D _MainTex;

  	struct a2v
  	{
  		float4 position : POSITION;
  		float2 uv : TEXCOORD0;
  	};

  	struct v2f
  	{
  		float4 position : SV_POSITION;
  		float2 uv : TEXCOORD0;
  	};

  	v2f vert(a2v v)
  	{
  		v2f o;
  		UNITY_INITIALIZE_OUTPUT(v2f, o);
  		o.position = UnityObjectToClipPos(v.position);
  		o.uv = v.uv;
  		return o;
  	}

  	half4 frag(v2f v) : SV_Target
  	{
  		half4 fragColor = half4(_Color.rgb,1.0) * tex2D(_MainTex, v.uv);
  		return fragColor;
  	}

  	ENDHLSL

  	SubShader
  	{		
  		Tags{ "Queue" = "Geometry" }
  		LOD 100
  		Pass
  		{
  			//注意这里,默认是没写光照类型的,自定义管线要求必须写,渲染脚本中会调用,否则无法渲染
  			//这也是为啥新建一个默认unlitshader,无法被渲染的原因
  			Tags{ "LightMode" = "Always" }
  			HLSLPROGRAM
  			#pragma vertex vert
  			#pragma fragment frag
  			ENDHLSL
  		}
  	}
  }
  ~~~

4. 加入平行光(Directional)
  - 在Shader中加入
  ```
  //平行光方向
  half4 _DLightDir;
  //平行光颜色
	fixed4 _DLightColor;
  struct a2v
  {
    ...
    //需要模型法线进行计算光照
		float3 normal : NORMAL;
    ...
  }
  struct v2f
  {
    ...
    float3 normal:NORMAL;
    ...
  }
  v2f vert(a2v v)
  {
    ...
    o.normal = UnityObjectToWorldNormal(v.normal);
    ...
  }
  half4 frag(v2f i)
  {
    ...
    //获得光照参数，进行兰伯特光照计算
		half light = saturate(dot(i.normal, _DLightDir));
    fragColor.rgb *= light * _DLightColor;
    ...
  }
  SubShader
  {
    ...
    Tags{"LightMode" = "BaseLit"}
    ...
  }
  ```

  - 在MyPipeline中加入
  ~~~
  //定义CommandBuffer用来传参
  private CommandBuffer _cb;
  protected override void Render(ScriptableRenderContext context, Camera[] cameras)
  {
    ...
    if(_cb = null) _cb = new CommandBuffer(name = "SRP Study CB");

    //将shader中需要的属性参数映射为ID，加速传参
    var _LightDir0 = Shader.PropertyToID("_DLightDir");
    var _LightColor0 = Shader.PropertyToID("_DLightColor");   
    for(var camera in cameras)
    {
        //设置渲染相关相机参数,包含相机的各个矩阵和剪裁平面等
        context.SetupCameraProperties(camera);               
        //清理_cb，设置渲染目标的颜色为灰色。
        _cb.ClearRenderTarget(true, true, Color.gray);  

        ...

        //在剪裁结果中获取灯光并进行参数获取
        var lights = cullResults.visibleLights;
        for(light in lights)
        {
          //判断灯光类型
          if (light.lightType != LightType.Directional) continue;
          //获取灯光参数,平行光朝向即为灯光Z轴方向。矩阵第一到三列分别为xyz轴项，第四列为位置。
          Vector4 lightpos = light.localToWorldMatrix.GetColumn(2);
          //灯光方向反向。默认管线中，unity提供的平行光方向也是灯光反向。光照计算决定
          Vector4 lightDir = -lightpos
          //方向的第四个值(W值)为0，点为1.
          lightDir.w = 0;
          //这边获取的灯光的finalColor是灯光颜色乘上强度之后的值，也正好是shader需要的值
          Color lightColor = light.finalColor;  
          //利用CommandBuffer进行参数传递。
          _cb.SetGlobalVector(_LightDir0, lightDir);
          _cb.SetGlobalColor(_LightColor0, lightColor);                    
          break;

        }  
        //执行CommandBuffer中的指令
        context.ExecuteCommandBuffer(_cb);
        _cb.Clear();

    }
    ...
  }
  ~~~

5. 加入高光(BlinPhone)
  - shader中加入
    ~~~
    Properties
    {
      ...
      _SpecularPow("SpecularPow",range(5,50)) = 20
      ...
    }

    //增加相机参数和高光强度
    half4 _CameraPos;
    fixed _SpecularPow;

    struct v2f
    {
      ...
      float3 worldPos : TEXCOORD1;
      ...
    }

    v2f vert(a2v v)
    {
    	...
    	o.worldPos = mul(unity_ObjectToWorld, v.position).xyz;
    	...
    }

    half4 frag(v2f i) : SV_Target
    {
      ...
      half3 viewDir = normalize(_CameraPos - i.worldPos);
  		half3 halfDir = normalize(viewDir + _DLightDir.xyz);
  		fixed specular = pow(saturate(dot(i.normal, halfDir)), _SpecularPow);
  		fragColor.rgb *= light * (1 + specular) * _DLightColor;
      ...
    }
    ~~~
  - 在MyPipeline中加入
    ~~~
    protected override void Render(ScriptableRenderContext context, Camera[] cameras)
    {
      ...
      var _CameraPos = Shader.PropertyToID("_CameraPos");
      ...

      foreach(camera in cameras)
      {
        ...
        Vector4 cameraPos = camera.transform.position;
        _cb.SetGlobalVector(_CameraPos, cameraPos);
        ...
      }
    }
    ~~~
6. 加入点光源(Point Light)
  https://zhuanlan.zhihu.com/p/102989404
7. 加入聚光灯(Spot Light)
  https://zhuanlan.zhihu.com/p/111691414
8. CommandBuffer的深入理解
  https://zhuanlan.zhihu.com/p/111691414
9. 平行光阴影(阴影计算原理)
  - 将灯光矩阵赋予相机，相机设定为正交相机。
  - 根据渲染相机的远近剪裁矩阵，调整灯光正交相机的Size至刚好包含整个可见区域。
  - 使用灯光正交相机绘制场景深度图。
  - 正常渲染场景时，转换场景到灯光相机屏幕空间，计算深度值并对灯光相机深度图进行采样。
  - 对比采样值与正常渲染时的深度值
  如果正常渲染时的深度值大（相机正向Z轴，越远，值越大）,就说明该位置相较于灯光位置更远，即该位置即为阴影区域。
10. 阴影计算的部分改进：
  - ShadowBias 因为深度图分辨率有限且非连续的原因，肯定会造成场景相互靠近的像素点共享深度图的同一个像素的情况，尤其是当灯光照射角度趋于水平的情况下。
  - 级联阴影 它是根据用户的分级距离设置，将渲染相机的视锥分段，然后用灯光相机分别匹配不同的视锥进行深度渲染，以生成多张深度图。通常使用atlas和偏移来储存生成的多张深度图。之后再根据设定的距离分级，对场景采样不同分辨率的深度图，这样就做到了根据不同视距匹配不同阴影分辨率的效果。
11. 在SRP中添加平行光阴影:
  - 投影平行光数量及阴影图格式：
    https://zhuanlan.zhihu.com/p/99142987



曲面细分或者置换材质

## 汇总
UnityEngine.Rendering 包中的类
1. CommandBuffer 要执行的图形命令的列表。
  //申请RT(RenderTexture)，后面参数分别为尺寸(宽、高)，深度缓存区位(0/16/24/32，如果渲染
  //结果不需要深度排序，则0最省。24/32则除了深度还会支持模板缓存区stencilbuffer)，
  //RT格式(为了性能，尽量取满足要求的最小尺寸格式，如R8,A8,RGB565等)
  rendertexture = new RenderTexture(256, 256, 0, RenderTextureFormat.RFloat);
  //清理命令缓存区，准备添加命令            
  buffer.Clear();
  //设置缓存区渲染目标
  buffer.SetRenderTarget(rendertexture);
  //清理渲染目标上的深度、颜色信息并设置默认色
  buffer.ClearRenderTarget(true, true, Color.gray);
  //常用的抓屏命令
  //将当前相机的渲染结果直接绘制到RT，可以指定渲染材质mat
  buffer.Blit(BuiltinRenderTextureType.CameraTarget, rendertexture, mat);
  //将指定的渲染器渲染到RT
  buffer.DrawRenderer(meshRender, mat);
2. CullingResults 剔除结果(可见对象、光源、反射探针)。在脚本化渲染循环中，渲染过程通常会对每个摄像机进行剔除(ScriptableRenderContext.Cull),然后渲染可见对象(ScriptableRenderContext.DrawRenderers)的子集并处理可见
3. DrawingSettings 描述如何对已排序的可见对象进行排序(sortingSettings)以及使用那些着色器通道(ShaderPassName) (ScriptableRenderContext.DrawRenderers的设置)
4. FilteringSettings 描述如何过滤给定的一组可见对象以便渲染。(ScriptableRenderContext.DrawRenderers的过滤设置)
5. RenderPipeline 定义一系列描述Unity如何渲染帧命令和设置
6. RenderPipelineAsset 生成特定IRenderPipeline的资源
7. ScriptableRenderContext 定义自定义渲染管线使用的状态和绘制命令，定义自定义RenderPipeline时，可使用ScriptableRenderContext 向GPU调度和提交状态更新和绘制命令。
8. ShaderTagId 着色器标签ID用于引用着色器中的各种名称。
9. SortingSettings 此结构描述在渲染期间对对象排序的方法。
10. VisibleLight 保留可见光源的数据



https://catlikecoding.com/unity/tutorials/

https://catlikecoding.com/unity/tutorials/scriptable-render-pipeline/custom-pipeline/

https://zhuanlan.zhihu.com/p/36407658

















11
