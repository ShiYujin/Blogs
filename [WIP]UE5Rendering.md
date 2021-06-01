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

它的作用是调用一个名为`EnqueueUniqueRenderCommand`的函数，这个函数定义在`Engine\Source\Runtime\RenderCore\Public\RenderingThread.h`:L233

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



