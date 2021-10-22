# Custom RP
## Custom Render Pipeline
  1. 项目设置
    - 设置Color Space 为Linear,使用Unlit/Color 创建不透明的物体,Unlit/Transparent 创建透明物体
    - 创建CustomRenderPipelineAsset
      ~~~
      using UnityEngine;
      using UnityEngine.Rendering;

      [CreateAssetMenu(menuName = "Rendering/Custom Render Pipeline")]
      public class CustomRenderPipelineAsset : RenderPipelineAsset
      {
          protected override RenderPipeline CreatePipeline()
          {
              return new CustomRenderPipeline();
          }
      }
      ~~~
    - 创建CustomRenderPipeline
      ~~~
      using UnityEngine;
      using UnityEngine.Rendering;

      public class CustomRenderPipeline : RenderPipeline
      {
          private CameraRenderer renderer = new CameraRenderer();
          protected override void Render(ScriptableRenderContext context, Camera[] cameras)
          {
              foreach (var camera in cameras)
              {
                  renderer.Render(context, camera);
              }
          }
      }
      ~~~
    - 创建CameraRenderer(partial)
      ~~~
      using System.Collections;
      using System.Collections.Generic;
      using UnityEngine;
      using UnityEngine.Rendering;

      public partial class CameraRenderer
      {
        //定义自定义渲染管线使用的状态和绘制命令
        private ScriptableRenderContext _context;
        //摄像机
        private Camera _camera;
        //渲染摄像机的名称
        private const string bufferName = "Render Camera";
        //要执行的图形命令的列表
        private CommandBuffer _buffer = new CommandBuffer {
            name = bufferName
        };
        //剔除结果
        private CullingResults _cullingResults;
        //正常Pass tagName
        static ShaderTagId unlitShaderTagId = new ShaderTagId("SRPDefaultUnlit");
        public void Render(ScriptableRenderContext context,Camera camera)
        {
            _context = context;
            _camera = camera;
            //开启Editor Only 、设置CommandBuffer名称
            PrepareBuffer();
            //Emits UI geometry into the Scene view for rendering.
            PrepareForSceneWindow();
            if (!Cull()) return;

            //设置摄像机属性、清除摄像机渲染目标(用CameraClearFlags来清除)
            Setup();
            //渲染可见几何体
            DrawVisibleGeometry();
            DrawUnsupportedShaders();
            DrawGizmos();
            Submit();
        }

        private void Setup()
        {
            //为Context设置Camera的属性
            _context.SetupCameraProperties(_camera);
            //设置Camera clear 标志
            CameraClearFlags flags = _camera.clearFlags;
            //清除摄像机渲染目标,清理深度Depth小于等于的,清理颜色不为Color标志的,当标志为Color的时候为背景颜色不然就是黑色
            _buffer.ClearRenderTarget(flags <= CameraClearFlags.Depth,
                flags == CameraClearFlags.Color,
                flags == CameraClearFlags.Color ?
                    _camera.backgroundColor.linear : Color.clear);

            // _buffer.ClearRenderTarget(flags <= CameraClearFlags.Depth, false,  Color.clear);
            _buffer.BeginSample(SampleName);

            ExecuteBuffer();

        }

        private void DrawVisibleGeometry()
        {
            //排序设置-默认排序标准(非透明物体)
            var sortingSettings = new SortingSettings(_camera)
            {
                criteria = SortingCriteria.CommonOpaque
            };

            //渲染设置-利用ShaderTag来选择渲染Pass
            var drawingSettings = new DrawingSettings(unlitShaderTagId, sortingSettings);
            //过滤设置-区分非透明物体
            var filteringSettings = new FilteringSettings(RenderQueueRange.opaque);

            //执行渲染-非透明物体
            _context.DrawRenderers(
                _cullingResults, ref drawingSettings, ref filteringSettings
            );

            //绘制Skybox
            _context.DrawSkybox(_camera);

            //排序设置-默认排序标准(透明物体)
            sortingSettings.criteria = SortingCriteria.CommonTransparent;
            drawingSettings.sortingSettings = sortingSettings;
            //过滤设置-透明物体
            filteringSettings.renderQueueRange = RenderQueueRange.transparent;

            //执行渲染-透明物体
            _context.DrawRenderers(
                _cullingResults, ref drawingSettings, ref filteringSettings
            );
        }

        /// <summary>
        /// 提交
        /// </summary>
        private void Submit()
        {
            _buffer.EndSample(SampleName);
            ExecuteBuffer();
            _context.Submit();
        }

        /// <summary>
        /// 执行CommandBuffer
        /// </summary>
        private void ExecuteBuffer()
        {
            _context.ExecuteCommandBuffer(_buffer);
            _buffer.Clear();
        }

        /// <summary>
        /// 剔除
        /// </summary>
        /// <returns></returns>
        private bool Cull()
        {
            if (_camera.TryGetCullingParameters(out ScriptableCullingParameters p))
            {
                _cullingResults = _context.Cull(ref p);
                return true;
            }

            return false;
        }
      }
      ~~~
    - 创建CameraRenderer.Editor(partial)
      ~~~
      using UnityEditor;
      using UnityEngine;
      using UnityEngine.Rendering;
      using UnityEngine.Profiling;
      partial class CameraRenderer
      {
          /// <summary>
          /// 渲染Gizmos
          /// </summary>
          partial void DrawGizmos();

          /// <summary>
          /// 渲染不支持的Shaders,错误提示灯
          /// </summary>
          partial void DrawUnsupportedShaders ();

          /// <summary>
          /// 渲染场景窗口中的geometry
          /// </summary>
          partial void PrepareForSceneWindow ();

          /// <summary>
          /// 为Profiler调试中设置Editor only
          /// 为FrameDebug中给buffer设置摄像机名称用来区分
          /// </summary>
          partial void PrepareBuffer();

      #if UNITY_EDITOR
          string SampleName { get; set; }
          partial void PrepareBuffer() {
              Profiler.BeginSample("Editor Only");
              _buffer.name = SampleName = _camera.name;
              Profiler.EndSample();
          }

          partial void PrepareForSceneWindow()
          {
              if (_camera.cameraType == CameraType.SceneView) {
                  ScriptableRenderContext.EmitWorldGeometryForSceneView(_camera);
              }
          }

          partial void DrawGizmos()
          {
              if (Handles.ShouldRenderGizmos()) {
                  _context.DrawGizmos(_camera, GizmoSubset.PreImageEffects);
                  _context.DrawGizmos(_camera, GizmoSubset.PostImageEffects);
              }
          }

          static ShaderTagId[] legacyShaderTagIds =
          {
              new ShaderTagId("Always"),
              new ShaderTagId("ForwardBase"),
              new ShaderTagId("PrepassBase"),
              new ShaderTagId("Vertex"),
              new ShaderTagId("VertexLMRGBM"),
              new ShaderTagId("VertexLM"),
              new ShaderTagId("BaseLit"),
          };

          //错误材质
          static Material errorMaterial;

          partial void DrawUnsupportedShaders()
          {
              if (errorMaterial == null)
              {
                  errorMaterial =
                      new Material(Shader.Find("Hidden/InternalErrorShader"));
              }

              var drawingSettings = new DrawingSettings(
                  legacyShaderTagIds[0], new SortingSettings(_camera)
              )
              {
                  overrideMaterial = errorMaterial
              };

              // var drawingSettings = new DrawingSettings(
              //     legacyShaderTagIds[0], new SortingSettings(_camera)
              // );

              for (int i = 1; i < legacyShaderTagIds.Length; i++)
              {
                  drawingSettings.SetShaderPassName(i, legacyShaderTagIds[i]);
              }

              var filteringSettings = FilteringSettings.defaultValue;
              _context.DrawRenderers(
                  _cullingResults, ref drawingSettings, ref filteringSettings
              );
          }
      #else

          const string SampleName = bufferName;

      #endif
      }

      ~~~
## Draw Calls
  - Shaders
    - Unlit Shader
      1. HLSL(Hight-Level Shading Language) 我们需要把他放在Pass中
      2. 为了绘制网格,GPU会光栅化所有三角形，需要将3D坐标转化为2D可视化像素,然后填充结果三角形，这两个步骤由2个着色器程序控制(vertex、fragment),片段着色器用于显示像素，尽管他可能会被后面所覆盖,必须添加vertex 与fragment的程序
      3. 引入UnlitPass.hlsl,UnlitPassVertex与UnlitPassFragment的实体写入hlsl
      ~~~
      Shader "Densefog/CustomRp/Unlit"{
          Properties{}
          SubShader{
            Pass{
                HLSLPROGRAM
                #pragma vertex UnlitPassVertex
                #pragma fragment UnlitPassFragment
                #include "UnlitPass.hlsl"
                ENDHLSL
            }
          }  
      }
      ~~~
    - UnlitPass.hlsl
      1. 加入hlsl保护,为了防止Shader代码重复
      2. 加入UnlitPassVertex与UnlitPassFragment空方法可以获得青色的默认材质
      3. 用0.0代替float4(0.0,0.0,0.0,0.0,0.0) SV_POSITION与SV_TARGET 返回值的语义,所有定点设置到0的时候所有定点会在同一个点，显示不出任何内容
      4. 使用参数需要设置语义(positionOS : POSITION 模型空间Object Space)
        - Common.hlsl 在ShaderLibrary文件夹中
          ~~~
          #ifndef DENSEFOG_CUSTOMRP_COMMON_INCLUDE
          #define DENSEFOG_CUSTOMRP_COMMON_INCLUDE
          #include "UnityInput.hlsl"

          float3 TransformObject2World(float3 positionOS){
              return mul(unity_ObjectToWorld,float4(positionOS,1.0)).xyz;
          }

          float4 TransformWorldToHClip (float3 positionWS) {
              return mul(unity_MatrixVP, float4(positionWS, 1.0));
          }

          #endif
          ~~~
        - UnityInput.hlsl 在ShaderLibrary文件夹中
          ~~~
          #ifndef DENSEFOG_CUSTOMRP_UNITY_INPUT_INCLUDE
          #define DENSEFOG_CUSTOMRP_UNITY_INPUT_INCLUDE

          float4x4 unity_ObjectToWorld;
          float4x4 unity_MatrixVP;

          #endif
          ~~~
        - UnlitPass.hlsl 在Shaders文件夹中
          ~~~
          #ifndef CUSTOM_UNLIT_PASS_INCLUDE
          #define CUSTOM_UNLIT_PASS_INCLUDE

          #include "../ShaderLibrary/Common.hlsl"

          float4 UnlitPassVertex (float3 positionOS : POSITION) : SV_POSITION {

              float3 positionWS = TransformObject2World(positionOS);
              return TransformWorldToHClip(positionWS);
          }

          float4 UnlitPassFragment () : SV_TARGET {

              return 0.0;
          }

          #endif
          ~~~
        - Unlit.shader
          ~~~
          Shader "Densefog/CustomRP/Unlit"{
          Properties{}
          SubShader{
                Pass{
                      //Tags{ "LightMode" = "SRPDefaultUnlit" }
                      HLSLPROGRAM
                      #pragma vertex UnlitPassVertex
                      #pragma fragment UnlitPassFragment
                      #include "UnlitPass.hlsl"
                      ENDHLSL
                }    
              }
          }
          ~~~
      5. 使用Core Library
        - 在Common.hlsl中的坐标系转换->"Packages/com.unity.render-pipelines.core/ShaderLibrary/SpaceTransforms.hlsl"
        - 在UnityInput.hlsl中加入->"Packages/com.unity.render.pipelines.core/ShaderLibrary/Common.hlsl" 存放了各种通用变量
      6.
  - Batching
    选择着色器，可以看到SRP Batcher not compatible, Material property is found in another cbuffer than "UnityPerMaterial"(_BaseColor)

## Driectional lights
## Directional Shadows
## Baked Light
## Shadow Masks
## LOD and Reflections
## Complex Maps
## Point and Spot lights
## Point and Spot Shadows
## Post Processing
## HDR
## Color Grading
## Multiple Cameras
## Particles
## Render Scale
## FXAA














































111
