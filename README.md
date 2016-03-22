# MyVideoPlayer
Very basic Video Player using AVFoundation


使用 AVPlayer 进行多视频播放
===================

> AVFoundation 框架提供了 AVPlayer 对象来实现单视频或多视频播放的控制器和用户接口。由 AVPlayer
> 对象生成的可视结果可以显示在 AVPlayerLayer 类的 CoreAnimation 层上。 在 AVFoundation
> 中，AVAsset
> 对象用来表示定时影音媒体，比如视频和音频。根据相关文档，每个资源包含一个用来一起呈现或处理的轨道集合，每个统一媒体类型，包括不仅限于音频、视频、文本、隐藏式字幕、字幕。因为定时影音媒体的性质，在成功初始化一个资源后，某些或全部键值可能不会立即可用。为了避免阻塞主线程，你可以在特定键注册你感兴趣的内容，以在可用时被通知到。
> 
> 考虑到这一点，继承 UIViewController 并命名为 VideoPlayerViewController。就像
> MPMoviePlayerController ，让我们添加一个 NSURL
> 属性，用于告诉我们应该从哪里抓取视频。就像上面描述的那样，添加下面的代码，一旦 URL 被赋值 AVAsset 就会被加载。

#pragma mark - Public methods  
 

    - (void)setURL:(NSURL*)URL {
          [_URL release];
          _URL = [URL copy];
          AVURLAsset *asset = [AVURLAsset URLAssetWithURL:_URL options:nil];
          NSArray *requestedKeys = [NSArray arrayWithObjects:kTracksKey,
                    kPlayableKey, nil];
          [asset loadValuesAsynchronouslyForKeys:requestedKeys
                               completionHandler: ^{ dispatch_async(
                                        dispatch_get_main_queue(), ^{
                                     [self prepareToPlayAsset:asset
     withKeys:requestedKeys];
                               });
          }];
    } 
    - (NSURL*)URL {
          return _URL;
    }

> 所以，一旦视频的 URL 被赋值后，创建一个资源来检查被指定的URL引用的源并且异步的加载这个资源的 “tracks” 和
> “playable” 键值。等加载结束后，我们就可以在主线程操作
> AVPlayer（当播放状态动态改变时，主线程可以确保安全的获取播放器的非原子属性）。

#pragma mark - Private methods 

    - (void)prepareToPlayAsset: (AVURLAsset *)asset withKeys: 
     (NSArray *)requestedKeys { 
         for (NSString *thisKey in requestedKeys) { 
            NSError *error = nil;  
            AVKeyValueStatus keyStatus = [asset  
                 statusOfValueForKey:thisKey
                               error:&amp;amp;error];  
            if (keyStatus == AVKeyValueStatusFailed) {  
               return;
            } 
         }
     if (!asset.playable) {
               return;
          }
          if (self.playerItem) {
              [self.playerItem removeObserver:self forKeyPath:kStatusKey];
              [[NSNotificationCenter defaultCenter] removeObserver:self 
                       name:AVPlayerItemDidPlayToEndTimeNotification
                     object:self.playerItem];
          }
          self.playerItem = [AVPlayerItem playerItemWithAsset:asset];
          [self.playerItem addObserver:self forKeyPath:kStatusKey 
                  options:NSKeyValueObservingOptionInitial |
                          NSKeyValueObservingOptionNew
                  context:
               AVPlayerDemoPlaybackViewControllerStatusObservationContext];
          if (![self player]) {
                [self setPlayer:[AVPlayer playerWithPlayerItem:self.playerItem]];
                [self.player addObserver:self forKeyPath:kCurrentItemKey 
                      options:NSKeyValueObservingOptionInitial |
                              NSKeyValueObservingOptionNew
                      context:
                 AVPlayerDemoPlaybackViewControllerCurrentItemObservationContext];
          }
          if (self.player.currentItem != self.playerItem) {
                 [[self player] replaceCurrentItemWithPlayerItem:self.playerItem];
          }
    }


> 在资源所有需要的键值加载完成后，我们检查是否加载成功以及该资源是否可以播放。如果这样，我们初始化一个 AVPlayerItem
> （用来表示能被 AVPlayer 对象播放的资源的表示状态）和一个 AVPlayer
> 来播放的资源。请注意，我在这一点上没有添加任何错误处理。在这里，我们应该创建一个委托并让视图控制器或正在使用你的播放器的用户，决定如何以最好的方式来处理可能出现的错误。
> 
> 我们也添加了一些键值监听以便于当我们的视图被绑定到播放器时和 AVPlayerItem 准备好播放时收到通知。

#pragma mark - Key Valye Observing

    - (void)observeValueForKeyPath: (NSString*) path
                          ofObject: (id)object
                            change: (NSDictionary*)change
                           context: (void*)context {
        if (context == AVPlayerDemoPlaybackViewControllerStatusObservation
                 Context) {
                  AVPlayerStatus status = [[change objectForKey:
                    NSKeyValueChangeNewKey] integerValue];
                  if (status == AVPlayerStatusReadyToPlay) {
                       [self.player play];
                  }
        } else if (context == AVPlayerDemoPlaybackViewControllerCurrentItem
                 ObservationContext) {
                  AVPlayerItem *newPlayerItem = [change objectForKey:
                     NSKeyValueChangeNewKey];
     
                  if (newPlayerItem) {
                      [self.playerView setPlayer:self.player];
                      [self.playerView setVideoFillMode:
                          AVLayerVideoGravityResizeAspect];
                  }
        } else {
            [super observeValueForKeyPath:path ofObject: object
                        change:change context:context];
        }
    }


> 一旦 AVPlayerItem 设置好后，我们可以自由的将 AVPlayer
> 添加到用来展示可视输出的播放器层。我们也会确保保留视频的长宽比和适合视频图层的边界内。
> 
> 一旦 AVPlayer 准备好了，就让它开始播放！让 iOS 来完成艰巨的任务 :)
> 
> 正如我前面所说，为了播放资源可视组件，您需要一个包含 AVPlayerLayer 的视图，来指挥 AVPlayer
> 对象的输出。下面演示了如何子类化 UIView 来满足要求︰

    @implementation VideoPlayerView
     
    + (Class)layerClass {
        return [AVPlayerLayer class];
    }
     
    - (AVPlayer*)player {
        return [(AVPlayerLayer*)[self layer] player];
    }
     
    - (void)setPlayer: (AVPlayer*)player {
        [(AVPlayerLayer*)[self layer] setPlayer:player];
    }
     
    - (void)setVideoFillMode: (NSString *)fillMode {
        AVPlayerLayer *playerLayer = (AVPlayerLayer*)[self layer];
        playerLayer.videoGravity = fillMode;
    }
     
    @end

