# Unity性能优化

## 第零讲 项目创建与总览

### 项目未优化时的一些参考指标数据

- 生成的Android APK大小 500M
- 三角形平面数 （Tris） 150-200万，峰值 230万
- 渲染批次（Batches） 1500-1800次
- SetPass calls 200次以上
- 小米11 ultra平均 FPS 10，iPhone XS Max 15
- 内存 小米11 ultra 1.5GB，iPhone XS Max 1GB
- 纹理资源小米11 ultra 670M，iPhone XS Max 530M
- Mesh资源小米11 ultra 423M，iPhone XS Max 423M
- 音效资源小米11 ultra 76M，iPhone XS Max 76M

## 第一讲 静态资源优化（1）——Audio导入设置检查与优化

- Unity资源工作流程与Asset checker的使用

- 资源检测报告——Audio部分问题解读
  - 根据平台选择合理的音频设置，原始音频资源尽量采用未压缩WAV格式
  - 移动平台对音乐音效统一采用单通道设置（Force To Mono），并将音乐采样频率设置为22050Hz
  - 移动平台大多数声音尽量采用Vorbis压缩设置，IOS平台或不打算循环的声音可以选择MP3格式，对于简短、常用的音效，可以采用解码速度快的ADPCM格式（PCM为未压缩格式）
  - 音频片段加载类型说明
    - 简短音效导入后小于200kb，采用Decompress On Load模式
    - 对于复杂音效，大小大于200kb，长度超过5秒的音效采用Compressed In Memroy模式
    - 对于长度较长的音频或背景音乐则采用Streaming模式，虽然会有CPU额外开销，但节省内存并且加载不卡顿
  - 当实现静音功能时，不要简单的将音量设置为0，应销毁（AudioSource）组件，将音频从内存中卸载。
- 优化前后真机对比（Android）
  - Apk包体由原来的560.7M下降到544.6M
  - Audio部分的内存由76.1M下降到6.9M
  - CPU开销由2.5%左右上升到5%左右，这是由于部分音频资源采用了Streaming模式加载

## 第二讲 静态资源优化（2）——Model导入设置检查与优化

- DCC中模型导出
  - Unity支持多种标准和专有模型文件格式（DCC）。Unity内部使用.fbx文件格式作为其导入链。最佳做法尽可能使用.fbx文件格式，并且不应在生产中使用专有文件格式
  - 优化原始导入模型文件，删除不需要的数据
    - 统一单位
    - 导出的网格必须是多边形拓扑网络，不能是贝塞尔曲线、样条曲线、NURBS、NURMS、细分曲面等
    - 烘焙Deformers，在导出之前，确保变形体被烘焙到网格模型上，如骨骼形变烘焙到蒙皮权重上
    - 不建议模型使用到的纹理随模型导出
    - 如果你需要导入blend shape normals，必须要指定光滑组smooth groups
    - DCC导出面板设置，不建议携带场景信息导出，如不建议导出摄像机、灯光、材质等信息。因为这些的信息与Unity内默认都不同。除非你自己为某DCC做过自定义导出插件
- Unity模型导入流程

![Unity模型导入流程](E:\Unity学习内容\性能优化相关\相关补充\Unity模型导入流程.jpg)

- 原始模型文件对性能的影响点
  - 最小化面数，不要使用微三角面（一个三角面内包含的像素过少），分布尽量均匀
  - 合理的网络拓扑和平滑组
  - 尽量少的使用材质个数
  - 尽可能少的使用蒙皮网络
  - 尽可能少的骨骼数量
  - FK与IK节点没分离，IK节点没删除
- 模型优化
  - 尽可能的将网络合并到一起
  - 尽可能使用共享材质
  - 不要使用网格碰撞体
  - 不必要不要开启网格读写
  - 使用合理的LOD级别
  - Skin Weights 受骨骼影响个数过多
  - 合理压缩网格
  - 不需要rigs和Blend Shapes尽量关闭
  - 如何可能，禁用法线或切线
  - 多套模型
- 资源检查报告——FBX部分问题解读
  - 其中两项建议与模型动画有关，而测试项目中所有模型资源都不涉及动画，可以将Rig标签下的Animation Type设置为None，并关闭Animation标签下的import Animations选项，设置Materials标签中的Material Creation Mode为None。
  - 开启Project Settings——>Player——>Optimizationa下的Vertex Compression与Optimize Mesh Data选项
- 优化结果
  - 包体大小前后没变化，依然是544.6M
  - 运行时模型内存优化前为422.9M，优化后为400.5M，相差的22M来自于运行时的CombinedMesh开销的节省。后期运行时模型资源优化才是我们的重点。

## 第三讲 静态资源优化（3）——纹理的基础概念

- 纹理类型

  - Default：默认的纹理类型格式
  - Normal map：发现贴图，可将颜色通道转换为适合实时法线贴图格式
  - Editor GUI and Legacy GUI：在编辑器GUI控件上使用纹理请选择此类型
  - Sprite（2D and UI）：在2D游戏中使用的精灵（Sprite）或UGUI使用的纹理请选择此类型
  - Cursor：鼠标光标自定义纹理类型
  - Cookie：用于光照Cookie剪影类型的纹理
  - Lightmap：光照贴图类型的纹理，编码格式取决于不同的平台
  - Single Channel：如果原始图片文件只有一个通道，请选择此类型

- 纹理大小

  - 选择合适纹理大小应尽量遵循以下经验：

    - 不同平台、不同硬件配置选择不同的纹理大小，Unity下可以采用bundle变体设置多套资源、通过Mipmap限制不同平台加载

      不同level层级的贴图

    - 根据纹理用途的不同选择不同的纹理加载方式，如流式纹理加载Texture Streaming、稀疏纹理Sparse Texture、虚拟纹理Virtual Texture等方式

    - 不能让美术人员通过增加纹理大小的方式增加细节，可以选择细节贴图DetailMap或增加高反差保留的方式

    - 在不降低视觉效果的情况下尽量减小贴图大小，最好的方式是纹理映射的每一个纹素的大小正好符合屏幕上显示像素的大小，如果纹理小了会造成欠采样，纹理显示模糊；如果纹理大了会造成过采样，纹理显示噪点。这一点做到完美很难保障，可以充分利用SceneView->DrawMode->Mipmap来查看在游戏摄像机视角下哪些纹理过采样，那些纹理欠采样来调整纹理大小。

- 纹理颜色空间

  - 默认大多数图像处理工具都会使用sRGB颜色空间处理和导出纹理。

    但如果你的纹理不是用作颜色信息的话，那就不要使用sRGB空间，

    如金属度贴图粗糙度贴图或者法线贴图等。

    一旦这些纹理使用sRGB空间会造成视觉表现错误。

- 纹理压缩

  - 纹理压缩是指图像压缩算法，保持贴图视觉质量的同时，尽量减小纹理数据的大小。

    默认情况下我们的纹理原始格式采用PNG或TGA这类通用文件格式，但与专用图像格式相比他们访问和采样速度都比较慢，无法通用GPU硬件加速，同时纹理数据量大，占用内存较高。

    所以在渲染中我们会采用一些硬件支持的纹理压缩格式，如ASTC、ETC、ETC2、DXT等。

- 纹理图集

  - 纹理图集是一系列小纹理图像的集合。
  - 优点：
    - 一是采用共同纹理图集的多个静态网格资源可以进行静态合批处理，减少DrawCall调用次数。
    - 二是纹理图集可以减少碎纹理过多，因为他们打包在一个图集里，通过压缩可以更有效的利用压缩，降低纹理的内存成本和冗余数据。
  - 缺点：
    - 美术需要合理规划模型，并且要求模型有相同的材质着色器，或需要制作通道图去区分不同材质。制作和修改成本较高。

- 纹理过滤

  - Nearset Point Filtering：临近点采样过滤最简单、计算量最小的纹理过滤形式，但在近距离观察时，纹理会呈现块状。
  - Bilinear Filtering：双线性采样过滤会对临近纹素采样并插值化处理，对纹理像素进行着色。双线性过滤会让像素看上去平滑渐变，但近距离观察时，纹理会变得模糊。
  - Trilinear Filtering：三线性过滤除了与双线性过滤相同部分外，还增加了Mipmap等级之间的采样差值混合，用来平滑过渡消除Mipmap之间的明显变化。
  - Anisotropic Filtering：各向异性过滤可以改善纹理在倾斜角度下的视觉效果，更适合用于地表纹理。

- 纹理Mipmap

  - Mipmap纹理
    - 逐级降低分辨率来保存纹理副本。相当于生成了纹理LOD，渲染纹理时，将根据像素在屏幕中占据的纹理空间大小选择合适的Mipmap级别进行采样。
  - 优点：
    - GPU不需要在远距离上对对象进行全分辨率纹理采样，因此可以提高纹理采样性能。
    - 同时也解决了远距离下的过采样导致的噪点问题，提高了纹理渲染质量。
  - 缺点：
    - 由于Mipmap纹理要生成低分辨率副本，会造成额外的内存开销。
## 第四讲 静态资源优化（4）——纹理导入设置检查与优化

- Texture Shape

  - 2D：最常用的2D纹理，默认选项。
  - Cube：一般用于天空盒与反射探针，默认支持Default、Normal、Single Channel几种类型纹理，可以通过 Assets > Create > Legacy > Cubemap 生成，也可以通过C#代码 Camera.RenderToCubemap 在脚本中生成。
  - 2D Array：2D纹理数组，可以极大提高大量相同大小和格式的纹理访问效率，但需要特定平台支持，可以通过引擎接口 SystemInfo.supports2DArrayTextures 运行时查看是否支持。
  - 3D：通过纹理位图方式存储或传递一些3D结构化数据，一般用于体积仿真，如雾效、噪声、体积数据、距离场、动画数据等信息，可以外部导入，也可运行时程序化创建。

- Alpha Source

  - 默认选择Input Texture Alpha就好，如果确定不使用原图中的Alpha通道，可以选择None。另外From Gray Scale一般不会选用。

- Alpha Is Transparency

  - 指定Alpha通道是否开启半透明，如果位图像素不关心是否要半透明可不开启此选项。这样Alpha信息只需要占1bit，节省内存。

- Ignore Png file gamma

  - 是否忽略png文件中的gamma属性，这个选项是否忽略取决于png文件中设置不同gamma属性导致的显示不正常，一般原图制作流程没有特殊设置，这个选项一般默认就好。

- Read/Write

- Streaming Mipmaps

- Virtual Texture Only

- Generate Mip Maps

  - 什么时候不需要生成Mipmaps
    - 2D场景
    - 固定视角，摄像机无法缩放远近
  - Border Mip Maps默认不开启，只有当纹理是Light Cookies类型时，开启此选项来避免colors bleeding现象导致颜色渗透到较低级别的Mip Level纹理边缘上
  - Mip Map Filtering
    - Box：最简单，随尺寸减小，Mipmap纹理变得平滑模糊
    - Kaiser：避免平滑模糊的锐化过滤算法
  - Mip Maps Preserve Coverage，只有需要纹理在开启mipmap后也需要做Alpha Coverage时开启。默认不开启。
  - Fadeout Mip Maps，纹理Mipmap随Mip层级淡化为灰色，一般不开启，只有在雾效较大时开启不影响视觉效果。

- 选择合适纹理过滤的最佳经验（放到导入设置）：

  - 使用双线性过滤平衡性能和视觉质量。
  - 有选择地使用三线性过滤，因为与双线性过滤相比，它需要更多的内存带宽。
  - 使用双线性和2x各向异性过滤，而不是三线性和1x各向异性过滤因为这样做不仅视觉效果更好，而且性能也更高。
  - 保持较低的各向异性级别。进队关键游戏资源使用高于2的级别。

- 其他常见的纹理问题：

  - 纹理图集利用率偏低，浪费内存
  - 生命周期类似的放在一起
  - 不合理的半透明纹理，造成Overdraw与内存开销
  - 不打图集的序列帧动画与过多的序列帧
  - 大量重复纹理与不合理的通道图
  - 内容相似贴图只是颜色变化
  - 大量颜色渐变贴图不压缩，也不采用单像素宽

## 第五讲 静态资源优化（5）——动画导入设置检查与优化

- Model里的相关设置

  - Animation Type

    - None：无动画
    - Legacy：旧版动画，不要用
    - Generic：通用骨骼框架
    - Hunmanoid：人形骨骼框架
    - 选择原则：
      - 无动画选择None
      - 非人型选择Generic
      - 人形动画：
        - 需要Kinematices或Animation Retargeting功能，或者有自定义骨骼对象时选择Humanoid Rig
        - 其他都选择Generic Rig，在骨骼数差不多的情况下，Generic Rig会比Humanoid Rig节省30%甚至更多的CPU时间

  - Skin Weights

    - 默认4根骨头，单对于一些不重要的动画对象可以减少到1根，节省计算量

  - Optimize Bones

    - 建议开启，在导入时自动剔除没有蒙皮顶点的骨骼

  - Optimize Game Objects

    - 在Avatar和Animator组件中删除导入游戏角色对象的变化层级结构，而使用Unity动画内部结构骨骼，消减骨骼transform带来的性能开销

    - 可以提高动画角色性能，但有些情况下会造成角色动画错误，这个选项可以尝试开启但要看表现效果而定。

      注意如果你的角色是可以换装的，在导入时不要开启此选项，但在换装后运行时可以在代码中通过调用。AnimatorUtility.OptimizeTransformHierarchy接口仍然可以达到此选项效果。

  - Resample Curves

    - 将动画曲线重新采样为四元数值，并为动画每帧生成一个新的四元数关键帧，仅当导入动画文件包含欧拉曲线时才会显示此选项。
    
  - Anim.Compression
      - Off：不压缩，质量最高，内存消耗最大。
      - Keyframe Reduction：减少冗余关键帧，减小动画文件大小和内存大小。
      - Optimal：仅适用于Generic与Humanoid动画类型，Unity决定如何进行压缩。
      
  - Animation Custom Properties
  
      - 导入用户自定义属性，一般对应DCC工具中的 extraUserProperties 字段中定义的数据。
  
- 动画曲线数据信息

  - Curves Pos：位置曲线
  - Quaternion：四元数曲线 Resample Curves开启会有
  - Euler：欧拉曲线
  - Scale：缩放曲线
  - Muscles：肌肉曲线，Humanoid类型下会有
  - Generic：一般属性动画曲线，如颜色，材质等
  - PPtr：精灵动画曲线，一般2D系统下会有
  - Curves Total：曲线总数
  - Constant：优化为常数的曲线
  - Dense：使用了密集数据（线性插值后的离散值）存储
  - Stream：使用了流式数据（插值的时间和切线数据）存储

- 动画文件导入设置优化后信息查看原则

  - 一：看效果差异（与原始制作动画差异是否明显）
  - 二：看曲线数量（总曲线数量与各种曲线数量，常量曲线比重大更好）
  - 三：看动画文件大小（动画文件在小几百k或更少合理，超过1M以上的动画文件考虑是否合理）

## 第六讲 Unity工作流（1）——工程目录与Assets目录设置

- Unity工程目录结构及用途
  - Assets文件夹：用来存储和重用的项目资产
  - Library文件夹：用来存储项目内部资产数据信息的目录
  - Packages文件夹：用来存储项目的包文件信息
  - Project Settings文件夹：用来存储项目设置的信息
  - UserSettings文件夹：用来存储用户设置信息
  - Temp文件夹：用来存储使用Unity编辑器打开项目时的临时数据，一旦关闭Unity编辑器也会被删除
  - Logs文件夹：用来存储项目的日志信息（不包含编辑器的日志信息）
- Assets目录结构设计
  - 一级目录设计原则：
    - 目录尽可能少
    - 区分编辑模式与运行模式
    - 区分工程大版本
    - 访问场景文件、全局配置文件便携
    - 不在一级目录做资源类别区分，只有Video类视频建议直接放到StreamingAssets下
  - 二级目录设计原则：
    - 只区分资源类型
    - 资源类型大类划分要齐全
    - 不做子类型区分
    - 不做功能区分
    - 不做生命周期区分
  - 三级目录设计原则：
    - Audio/Texture/Models三级目录做子类型区分
    - 其他模块类型资源做功能模块/生命周期区分
  - 四级目录设计原则：
    - 只有Audio/Texture/Models做四级目录，按模块/生命周期划分

## 第七讲 Unity工作流（2）——资源导入工作流

- 资源导入工作流的三种方案：
  - 1.手动编写工具
    - 优点：根据项目特点自定义安排导入工作流，并且可以和后续资源制作与打包工作流结合
    - 缺点：存在开发和维护成本，会让编辑器菜单界面变得复杂，对新人理解工程不友好
    - 适合类型：大型商业游戏团队
  - 2.利用Presets功能
    - 优点：使用简单方便，只需要Assets目录结构合理规范即可
    - 缺点：无法和后续工作流整合，只适合做资源导入设置
    - 适合类型：小型团队或中小规模项目
  - 3.利用AssetGraph工具
    - 优点：功能全，覆盖Unity资源工作流全流程，节点化编程，直观
    - 缺点：有一定上手成本，一些自定义生成节点也需要开发，不是Unity标准包，Unity新功能支持较慢
    - 适合类型：任何规模项目和中大型团队

## 第八讲 编辑器创建资源优化（1）——场景

- 场景结构设计原则：
  - 1.合理设计场景一级节点的同时，避免场景节点深度太深，一些代码生成的游戏对象如果不需要随父节点进行Transform的，一律放在根节点下。
  - 2.尽量使用Prefab节点构建场景，而不是直接创建的GameObject节点（因为物体的数据信息存的是Prefab的引用）。
  - 3.避免DontDestroyOnLoad节点下太多生命周期太长或引用资源过多的复杂节点对象。Additive场景尤其要注意。
  - 4.最好为一些需要经常访问的节点添加tag（使用FindGameObjectWithTag查找），静态节点一定要添加Static标记。
  - 注：有些工具可以提高开发时的性能（如3D World Building的包等）。

## 第九讲 编辑器创建资源优化（2）——预制体

- 使用预制体的好处
  - 由于预制体系统可以自动保持所有实例副本同步，因此可以比单纯地简单复制粘贴游戏对象做到更好的对象管理。
  - 此外通过预制体嵌套（Nested Prefabs）可以将一个预制体嵌套到另一个预制体中，从而创建多个易于编辑的复杂游戏对象层级视图。
  - 可以通过覆盖各个预制体实例的设置来创建预制体变体（Prefabs Variant），从而可以将一系列覆盖组合在一起形成有意义的预制体的变化。
- 嵌套预制体与单预制体相比的优点与缺点
  - 优点：
    - 嵌套预制体方便预制体管理，方便资源重复利用，易于统计场景复杂度
    - 美术制作时可以比较合理的分配UV和贴图利用率
    - 方便关卡设计人员发挥，充分合理利用资源
    - 比较方便利用工具做LOD，LOD效果也比较好
    - 修改方便，只需替换子预制体就可以做到所有预制体同步
    - 比较方便做场景遮挡剔除，可以做到精细的遮挡剔除优化效果
  - 缺点：
    - 手动做Bundle依赖时要按Scene方式处理，依赖关系较为复杂
    - 可能会增加材质数量与Drawcall数量
    - 不太适合做大规模远景对象
    - 美术与关卡设计人员要充分考虑组合复杂度与特例场景显示，避免重复性和单一性，需要更多的沟通成本
- 使用Prefab变体的一些限制
  - 不能改变本体Prefab游戏对象（GameObject）层级
  - 不能删除本体Prefab中的游戏对象，但可以通过Deactive游戏对象来达到与删除游戏对象同样的效果
  - 对于Prefab变体要保持其Override属性的变化，不能通过Apply to base把这些变化应用到本体Prefab，这样会破坏基础Prefab的结构和功能

## 第十讲 编辑器创建资源优化（3）——UGUI（上）

- Unity UI性能的四类问题
  - Canvas Re-batch 时间过长
  - Canvas Over-dirty，Re-batch次数过多
  - 生成网格顶点时间过长
  - Fill-rate overutilization
- Canvas画布
  - Canvas负责管理UGUI元素，负责UI渲染网格生成与更新，并向GPU发送DrawCall指令
- Canvas Re-batch过程
  - 1.根据UI元素深度关系进行排序
  - 2.检查UI元素的覆盖关系
  - 3.检查UI元素材质并进行合批
- UGUI渲染细节
  - UGUI中渲染是在Transparent半透明渲染队列中完成的，半透明队列的绘制顺序是从后往前，由于UI元素做Alpha Blend，我们在做UI时很难保障每一个像素不被重画，UI的Overdraw太高，这会造成片元着色器利用率过高，造成GPU负担。
  - UI SpriteAtlas图集利用率不高的情况下，大量完全透明的像素被采样也会导致像素被重绘，造成片元着色器利用率过高；同时纹理采样器浪费了大量采样在无效的像素上，导致需要采样的图集像素不能尽快的被采样，造成纹理采样器的填充率过低，同样也会带来性能问题。
- Re-build过程
  - 在WillRenderCanvases事件调用PerformUpdate::CanvasUpdateRegistry接口
    - 通过ICanvasElement.Rebuild方法重新构建Dirty的Layout组件
    - 通过ClippingRegistry.Cullf方法，任何已注册的裁剪组件Clipping Components（Such as Masks）的对象进行裁剪剔除操作
    - 任何Dirty的Graphics Components都会被要求重新生成图形元素
  - Layout Rebuild
    - UI元素位置、大小、颜色发生变化
    - 优先计算靠近Root节点，并根据层级深度排序
  - Graphic Rebuild
    - 顶点数据被标记成Dirty
    - 材质或贴图数据被标记成Dirty
- 使用Canvas的基本准则：
  - 1.将所有可能打断合批的层移到最下边的图层，尽量避免UI元素出现重叠区域
  - 2.可以拆分使用多个同级或嵌套的Canvas来减少Canvas的Rebatch复杂度
  - 3.拆分动态和静态对象放到不同Canvas下
  - 4.不使用Layout组件
  - 5.Canvas的RenderMode尽量Overlay模式，减少Camera调用的开销
- UGUI射线（Raycaster）优化
  - 必要的需要交互UI组件才开启“Raycast Target”
  - 开启“Raycast Targets”的UI组件越少，层级越浅，性能越好
  - 对于复杂的控件，尽量在根节点开启“Raycast Target”
  - 对于嵌套的Canvas，OverrideSorting属性会打断射线，可以降低层级遍历的成本

## 第十一讲 编辑器创建资源优化（4）——UGUI（下）

- UI字体
  - 优化注意
    - 避免字体框重叠，造成合批打断
    - 字体网格重建
      - UIText组件发生变化时
      - 父级对象发生变化时
      - UI组件或其父对象enable/disable时
  - TrueTypeFontImporter
    - 支持TTF和OTF字体文件格式导入
  - 动态字体与字体图集
    - 运行时，根据UIText组件内容，动态生成字体图集，只会保存当前Actived状态的UIText控件中的字符
    - 不同的字体库维护不同的Texture图集
    - 字体Size、大小写、粗体、斜体等各种风格都会保存在不同的字体图集中（有无必要，影响图集利用效率，一些利用不多的特殊字体可以采用图片代替或使用Custom Font，Font Assets Creater创建静态字体资源）
    - 当前Font Texture不包含UIText需要显示的字体时，当前Font Texture需要重建
    - 如果当前图集太小，系统也会尝试重建，并加入需要使用的字形，文字图集只增不减
    - 利用Font.RequestCharacterInTexture可以有效降低启动时间
- UI控件优化注意事项
  - 不需要交互的UI元素一定要关闭Raycast Target选项
  - 如果是较大的背景图的UI元素建议也要使用Sprite的九宫格拉伸处理，充分减小UI Sprite大小，提高UI Atlas图集利用率
  - 对于不可见的UI元素，一定不要使用材质的透明度控制显隐，因为那样UI网格依然在绘制，也不要采用active/deactive UI控件进行显隐，因为那样会带来gc和重建开销
  - 使用全屏的UI界面时，要注意隐藏其背后的所有内容，给GPU休息机会
  - 在使用非全屏但模态对话框时，建议使用OnDemandRendering接口，对渲染进行降频
  - 优化裁剪UI Shader，根据实际使用需求移除多余特性关键字
- 滚动视图Scroll View
  - 使用RectMask2d组件裁剪
  - 使用基于位置的对象池作为实例化缓存

## 第十二讲 编辑器创建资源优化（4）——物理

- Project Setting
  - Physics
    - Layer Collision Matrix
    - Auto Sync Transforms：勾选后物理对象变化后强行进行物理系统同步（立即更新），建议不勾选
    - Reuse Collision Callbacks：碰撞回调时会重用之前的Collision碰撞回调实例，而不会重新创建碰撞结果（建议保持开启）
    - Default Solver Iterations：物理结算器迭代次数设置（物理碰撞精度）
    - Default Solver Velocity Iterations ：物理结算器迭代次数设置（碰撞后物理模拟精度）
  - Time
    - Fixed Timestep：FixUpdate的更新频率
    - Maximum Allowed Timestep：限制FixUpdate的最大步长
- Unity中的物理组件Collider部分的优化
  - Trigger与Collider
    - Trigger对象的碰撞会被物理引擎所忽略，通过OnTriggerEnter/Stay/Exit函数回调
    - Collider对象由物理引擎触发碰撞，通过OnCollisionEnter/Stay/Exit函数回调
    - Trigger对象不需要RigidBody组件，Collider对象必须至少有一个Collider对象有RigidBody组件
    - Trigger对象更高效
  - Mesh Collider
    - 尽量少用MeshCollider，可以用简单Collider代替，即使用多个简单Collider组合代替也要比复杂的MeshCollider来的高效
    - MeshCollider是基于三角形面的碰撞
    - MeshCollider生成的碰撞体网格占用内存也较高
    - MeshCollider即使要用也要尽量保障其是静态物体
    - 可以通过PlayerSetting选项中勾选Prebake Collision Meshes选项来在构建应用时预先Bake出碰撞网格
- Unity中的物理组件RigidBody部分的优化
  - Kinematic与RigidBody
    - Kinematic对象不受物理引擎中力的影响，但可以对其他RigidBody施加物理影响
    - RigidBody完全由物理引擎模拟来控制，场景中RigidBody数量越多，物理计算负载越高
    - 勾选了Kinematic选项的RigidBody对象会被认为是Kinematic的，不会增加场景中的RigidBody个数
    - 场景中的RigidBody对象越少越好
- Unity中的RayCast与Overlap部分的优化
  - Unity物理中RayCast与Overlap都有NoAlloc版本的函数，在代码中调用时尽量用NoAlloc版本，这样可以避免不必要的GC开销
  - 尽量调用RayCast与Overlap时要指定对象图层进行对象过滤，并且RayCast要还可以指定距离来减少一些太远的对象查询
  - 此外如果是大量的RayCast操作还可以通过RaycastCommand的方式批量处理，充分利用JobSystem来分摊到多核多线程计算

## 第十三讲 编辑器创建资源优化（5）——动画

- Unity动画系统回顾

  ​	![](E:\Unity学习内容\性能优化相关\相关补充\Unity动画系统回顾.jpg)

  - Animation
    - 播放单个（注意：单个）Animation Clip速度，Legacy Animation系统更快，因为老系统是直接采样曲线并直接写入对象Transform
    - 针对动画的缩放曲线比位移、旋转矩阵开销更大
    - 常数曲线不会每帧写入场景，更高效
  - Animator

    - 不要使用字符串来查询Animator
    - 使用曲线标记来处理动画事件
    - 使用Target Marching函数来协助处理动画
    - 将Animator的CullingMode设置成Based On Renderers来优化动画，并禁用SkinMesh Renderer的Update When Offscreen属性来让角色不可见时动画不更新
  - Playable API

    - Playables API是一套树形结构来组织数据源，并允许用户通过脚本来创建和播放自定义的行为，支持与动画系统、音频系统等其他系统交互，是一套通用的接口

    - [PlayableGraph Visualizer]: https://github.com/UnityTech/graph-visualizer

      

- Animator VS Animation

  - Animation 可以将任何对象属性制作成Animation Clip，Animator是将Animation Clip组织到状态机流程图中使用

  - Animation与Animator播放动画时的效率是有个临界点的，这个临界点是根据动画曲线条数来的，当动画曲线小于这个临界点时Animation快，大于这个临界点时Animator快

  - 当CPU核数较少时，Animation播放动画有优势，当CPU核数较多时，Animator表现会更好

  - Animator Controller Graph中的所有动画节点的Animation Clip都会载入到内存中。当有海量动画状态机节点时，内存开销较大

- Playable API VS Animator

  - Playable API优点
    - 支持动态动画混合，可为场景中的对象提供自己的动画，并可以动态添加到PlayableGraph当中使用

    - 允许创建播放单个动画，而并不会产生创建和管理AnimatorController资源所涉及的开销，可更灵活的控制PlayableGraph的数据流，可以插入自定义的AnimationJob

    - 可以控制动画文件加载策略，按需加载、异步加载等

    - 允许用户动态创建混合图，并直接逐帧控制混合权重（甚至可以混合AnimationClip与AnimatorController动画）

    - 可以运行时动态创建，根据条件添加可播放节点。而不需要提前提供一套PlayableGraph运行时启动和禁用节点，可以做到自由度更高的override机制

    - 可加载自定义配置数据，更加方便的和其他游戏系统整合

  - Playable API缺点
    - 没有直接使用Animator直观
    - 混合模式没有现成的，需要自己实现
    - 需要开发更多的配套工具
    - 有一定的学习成本

- PlayableDirector与Timeline Asset的关系

- Animator Layer Mixer

- 解决方案选择

  - 一些简单、少量曲线动画可以使用Animation或动画区间库如Dotween、iTween等完成，如UI动画，Transform动画等
  - 角色骨骼蒙皮动画如果骨骼较少，Animation Clip资源不多，对动画混合表现要求不高的项目可以采用Legacy Animation。注意控制总体曲线数量
  - 一些角色动画要求与逻辑有较高交互、并且动画资源不多的项目可以直接用Animator Graph完成
  - 一些动作游戏，对动画混合要求较高、有一些高级动画效果要求、动画资源量庞大的项目，建议采用Animator + Playable API扩展Timeline的方式完成

- [Order of execution for event functions]: https://docs.unity3d.com/Manual/ExecutionOrder.html

## 第十四讲 特别篇——性能优化之道

- 性能优化问题的本质
  - 慢与快的问题
  - 前提：
    - 稳定性：不能因优化造成稳定性变差
    - 兼容性：不能因优化导致兼容性变差
    - 性价比：优化要有度，考虑成本与复杂度
- 性能优化的流程
  - 发现问题
    - 什么平台、什么操作系统、什么情况下出现问题，一般问题还是特例问题
  - 定位问题
    - 什么地方造成的性能问题，我们要用什么工具、什么方法确定瓶颈
  - 研究问题
    - 确定用什么方案处理这个问题，要考虑性能优化的前提
  - 解决问题
    - 按问题研究的结论去实际处理，并验证处理结果与预期的一致性
- 影响性能的四大类问题
  - CPU（工厂）
  - GPU（工厂）
  - 带宽（输送道路）
  - 内存（原材料厂）
- 隐藏的几类小问题
  - 功耗比
  - 填充率
  - 发热量
- 性能问题可能的情况
  - 瓶颈可能性按由高到低的顺序排列（个人经验总结）
    - CPU利用率
    - 带宽利用率
    - CPU/GPU强制同步
    - 片元着色器指令
    - 几何图形到CPU到GPU的传输
    - 纹理CPU到GPU的传输
    - 顶点着色器指令
    - 几何图形复杂性
- 经常用的优化思路
  - 升维与降维
  - 维度转换，空间与时间，量纲转换

  ## 第十五讲 性能优化实战（1）——性能总览与瓶颈定位

- 移动平台优化顺序
  - 先优化IOS，再优化Android
  - 先共性性能优化，再进行兼容性方面的性能优化
- Unity下常见的等待函数
  - WaitForTargetFPS：等待达到目标帧率，一般这种情况CPU与GPU都没什么负载问题
  - Gfx.WaitForGfxCommandsFromMainThread/WaitForCommand：渲染线程已经准备接受新的渲染命令，一般瓶颈在CPU
  - Gfx.WaitForPresentOnGfxThread/WaitForPresent：主线程等待渲染线程绘制完成，一般瓶颈在GPU
  - WaitForJobGroupID：等待工作线程完成，一般瓶颈在CPU
- 示例工程瓶颈判断
  - GPU与带宽可能是主要瓶颈
    - 渲染流程与效果优化
    - 渲染中生成资源优化
    - DrawCall与SetPassCall
    - 片元着色器
    - 渲染三角形

## 第十六讲 性能优化实战（2）——渲染流程分析

- 设备性能抓取工具
  - IOS
    - Run-Options-Metal，勾选Profile GPU Trace after capture
  - Android
    - Nvidia-NsightGraphics

## 第十七讲 性能优化实战（3）——SSAO优化

- 主要优化思路
  - 减少采样频率
  - 减少模糊的Pass
- SSAO进一步优化
  - 1.使用HBAO（horizon-based ambient occlusion）或GTAO（Ground Truth Ambient Occlusion）方案替代SSAO
  - 2.针对SSAO的Shader指令做进一步优化
  - 3.可以采用烘焙AO到光照贴图的方案替换SSAO方案

