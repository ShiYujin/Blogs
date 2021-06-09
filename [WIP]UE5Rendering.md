# Rendering in UE5

入口：`Engine\Source\Runtime\Engine\Private\GameEngine.cpp`:L1683:`void UGameEngine::Tick( float DeltaSeconds, bool bIdleMode )`:L1921`RedrawViewports();`

调用`UGameEngine`的渲染函数：`RedrawViewports();`：

`Engine\Source\Runtime\Engine\Private\GameEngine.cpp`:L693:`void UGameEngine::RedrawViewports( bool bShouldPresent /*= true*/ )`:L702:`GameViewport->Viewport->Draw(bShouldPresent);`

`GameViewport`是一个`UGameViewportClient`类，它的成员`Viewport`是一个`FViewport*`，`Viewport`的`Draw()`函数定义在：

`Engine\Source\Runtime\Engine\Private\UnrealClient.cpp`:L1472:`void FViewport::Draw( bool bShouldPresent /*= true */)`

接下来在L1562调用`FViewportClient* ViewportClient;`的`Draw`函数

其中3D绘制的代码在L3911:`GetRendererModule().BeginRenderingViewFamily(Canvas,&ViewFamily);`

`GetRendererModule()`的函数实现在`Engine\Source\Runtime\Engine\Private\EngineGlobals.cpp`:L69:

```C++
ENGINE_API IRendererModule& GetRendererModule()
{
	if (!CachedRendererModule)
	{
		CachedRendererModule = &FModuleManager::LoadModuleChecked<IRendererModule>(TEXT("Renderer"));
	}

	return *CachedRendererModule;
}
```

返回的`IRendererModule`其实是指向子类的一个指针，这个子类定义在`Engine\Source\Runtime\Renderer\Private\RendererModule.h`:L30:`class FRendererModule final : public IRendererModule`

`BeginRenderingViewFamily`函数定义在`Engine\Source\Runtime\Renderer\Private\SceneRendering.cpp`:L3712

`BeginRenderingViewFamily`这个函数会初获取渲染场景`FScene* const Scene`，然后将渲染命令入队（enqueue）。关键代码在`Engine\Source\Runtime\Renderer\Private\SceneRendering.cpp`:L3816-L3824：

```C++
		ENQUEUE_RENDER_COMMAND(FDrawSceneCommand)(
			[SceneRenderer, DrawSceneEnqueue](FRHICommandListImmediate& RHICmdList)
			{
				const float StartDelayMillisec = FPlatformTime::ToMilliseconds(FPlatformTime::Cycles() - DrawSceneEnqueue);
				CSV_CUSTOM_STAT_GLOBAL(DrawSceneCommand_StartDelay, StartDelayMillisec, ECsvCustomStatOp::Set);

				RenderViewFamily_RenderThread(RHICmdList, SceneRenderer);
				FlushPendingDeleteRHIResources_RenderThread();
			});

```

这里的宏`ENQUEUE_RENDER_COMMAND`定义在`Engine\Source\Runtime\RenderCore\Public\RenderingThread.h`:L263

```C++
#define ENQUEUE_RENDER_COMMAND(Type) \
	struct Type##Name \
	{  \
		static const char* CStr() { return #Type; } \
		static const TCHAR* TStr() { return TEXT(#Type); } \
	}; \
	EnqueueUniqueRenderCommand<Type##Name>

```

Q:`Name`是什么东西？
A:

`ENQUEUE_RENDER_COMMAND`的作用是调用一个名为`EnqueueUniqueRenderCommand`的函数，这个函数定义在`Engine\Source\Runtime\RenderCore\Public\RenderingThread.h`:L233

```C++
template<typename TSTR, typename LAMBDA>
FORCEINLINE_DEBUGGABLE void EnqueueUniqueRenderCommand(LAMBDA&& Lambda)
{
	//QUICK_SCOPE_CYCLE_COUNTER(STAT_EnqueueUniqueRenderCommand);
	typedef TEnqueueUniqueRenderCommandType<TSTR, LAMBDA> EURCType;

#if 0 // UE_SERVER && UE_BUILD_DEBUG
	UE_LOG(LogRHI, Warning, TEXT("Render command '%s' is being executed on a dedicated server."), TSTR::TStr())
#endif
	if (IsInRenderingThread())
	{
		FRHICommandListImmediate& RHICmdList = GetImmediateCommandList_ForRenderCommand();
		Lambda(RHICmdList);
	}
	else
	{
		if (ShouldExecuteOnRenderThread())
		{
			CheckNotBlockedOnRenderThread();
			TGraphTask<EURCType>::CreateTask().ConstructAndDispatchWhenReady(Forward<LAMBDA>(Lambda));
		}
		else
		{
			EURCType TempCommand(Forward<LAMBDA>(Lambda));
			FScopeCycleCounter EURCMacro_Scope(TempCommand.GetStatId());
			TempCommand.DoTask(ENamedThreads::GameThread, FGraphEventRef());
		}
	}
}
```

再回头看`BeginRenderingViewFamily`的这个宏调用，相当于：

```C++
	if (IsInRenderingThread())
	{
        FRHICommandListImmediate& RHICmdList = GetImmediateCommandList_ForRenderCommand();
        
        // 以下语句等价于 Lambda(RHICmdList);
        const float StartDelayMillisec = FPlatformTime::ToMilliseconds(FPlatformTime::Cycles() - DrawSceneEnqueue);
        CSV_CUSTOM_STAT_GLOBAL(DrawSceneCommand_StartDelay, StartDelayMillisec, ECsvCustomStatOp::Set);

        RenderViewFamily_RenderThread(RHICmdList, SceneRenderer);
        FlushPendingDeleteRHIResources_RenderThread();
	}
    else
    {
        ...
    }
```

这里面，我们关注的是函数`RenderViewFamily_RenderThread`调用。

`RenderViewFamily_RenderThread`函数定义在`Engine\Source\Runtime\Renderer\Private\SceneRendering.cpp`:L3493，它的作用主要是调用`SceneRenderer`的`SceneRenderer->Render(GraphBuilder);`函数。

第一个参数，`FRHICommandListImmediate RHICmdList`，它是一个命令列表，用于执行渲染命令。

第二个参数，`FSceneRenderer* SceneRenderer`，类`FSceneRenderer`定义在`Engine\Source\Runtime\Renderer\Private\SceneRendering.h`:L1722，s是一个渲染的基类，唯一的作用就是渲染场景。它由`FSceneViewFamily::BeginRender`初始化，并传递给渲染进程，通过调用`Render()`函数执行渲染操作，最后在渲染结束时被删除掉。

`FSceneRenderer* SceneRenderer`创建在：`Engine\Source\Runtime\Renderer\Private\SceneRendering.cpp`:L3783:

```C++
FSceneRenderer* SceneRenderer = FSceneRenderer::CreateSceneRenderer(ViewFamily, Canvas->GetHitProxyConsumer());
```

`CreateSceneRenderer`函数在：`Engine\Source\Runtime\Renderer\Private\SceneRendering.cpp`:L3260:

```C++
FSceneRenderer* FSceneRenderer::CreateSceneRenderer(const FSceneViewFamily* InViewFamily, FHitProxyConsumer* HitProxyConsumer)
{
	EShadingPath ShadingPath = InViewFamily->Scene->GetShadingPath();
	FSceneRenderer* SceneRenderer = nullptr;

	if (ShadingPath == EShadingPath::Deferred)
	{
		SceneRenderer = new FDeferredShadingSceneRenderer(InViewFamily, HitProxyConsumer);
	}
	else 
	{
		check(ShadingPath == EShadingPath::Mobile);
		SceneRenderer = new FMobileSceneRenderer(InViewFamily, HitProxyConsumer);
	}

	return SceneRenderer;
}
```

到这里，终于有图形学相关的名词出现了，根据`ShadingPath`的不同定义，`CreateSceneRenderer`会返回延迟渲染管线渲染器`FDeferredShadingSceneRenderer`或者用于移动设备的前向渲染管线渲染器`FMobileSceneRenderer`。

我们比较关心的是延迟管线，它定义在`Engine\Source\Runtime\Renderer\Private\DeferredShadingRenderer.h`:L163，继承自`FSceneRenderer`:

```C++
class FDeferredShadingSceneRenderer : public FSceneRenderer{
	...
}
```




