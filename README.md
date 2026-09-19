# 枫音 Android（FengYin Music）

依据《枫音音乐源码.html》的网页版逻辑，用 **Kotlin + Jetpack Compose + Media3(ExoPlayer)** 重写的原生 Android 音乐播放器。

---

## 1. 网页源码 → Android 实现映射

| 网页版数据流 | Android 实现 | 所在文件 |
| --- | --- | --- |
| `searchMusic()` 请求酷我搜索接口 | `MusicRepository.search(keyword, page)` → `KuwoApi.search` | `data/repository/MusicRepository.kt`、`data/remote/KuwoApi.kt` |
| `res.abslist` 映射成 `playlist` | `KuwoParser.parseSearch()` → `SearchPage(songs, isEnd)` | `data/remote/KuwoParser.kt` |
| 渲染歌曲列表 / 分页 | `HomeScreen` + `LazyColumn` + 上一页/下一页 | `ui/home/HomeScreen.kt` |
| 分页终止条件 `(PN+1)*RN >= TOTAL` | `KuwoParser` 计算 `isEnd`，对应 `HomeUiState.isEnd` | `data/remote/KuwoParser.kt` |
| `playSong()` 调第三方解析接口拿音频直链 | `MusicRepository.playUrl(musicId, quality)` | `data/repository/MusicRepository.kt` |
| `audioPlayer.src = url` 播放 | `MusicPlayer.play(song, url)` → ExoPlayer `setMediaItem/prepare/play` | `player/MusicPlayer.kt` |
| 歌词接口 + 渲染歌词 | `MusicRepository.lyrics()` → `KuwoParser.parseLyrics()` → `LyricLine` 列表 | `data/repository/MusicRepository.kt` |
| `ontimeupdate` 更新进度条 / 时间 | `MusicPlayer` 每 200ms 轮询写入 `StateFlow<PlayerState>` | `player/MusicPlayer.kt` |
| 计算当前歌词行 + 高亮 + 滚动 | `currentLyricIndex = lyrics.indexOfLast { it.timeMs <= positionMs }` + `animateScrollToItem` | `ui/player/PlayerScreen.kt` |
| 迷你歌词（底部浮条） | `MiniPlayer` 常驻底部导航上方 | `ui/components/MiniPlayer.kt` |
| `audioPlayer.onended` | `Player.STATE_ENDED` → `onSongFinished` → 自动下一首 | `player/MusicPlayer.kt`、`ui/MainViewModel.kt` |
| 上/下一首按钮 | `playPrev()` / `playNext()`（列表首尾循环） | `ui/MainViewModel.kt` |
| `QUALITY_MAP`（128k/320k/flac） | `enum Quality { STANDARD, HIGH, LOSSLESS }`，切换后按原进度续播 | `data/model/Quality.kt` |
| 高清封面 `songinfo.pic` 回填 | `highResArtwork()` → `MusicPlayer.updateArtwork()` | `data/repository/MusicRepository.kt` |

## 2. 工程结构

```
fengyin-android/
├── settings.gradle.kts / build.gradle.kts / gradle.properties
├── gradle/wrapper/gradle-wrapper.properties
└── app/
    ├── build.gradle.kts / proguard-rules.pro
    └── src/main/
        ├── AndroidManifest.xml
        ├── java/com/fengyin/music/
        │   ├── FengYinApp.kt                # Application，初始化播放器
        │   ├── MainActivity.kt              # 入口 Activity（通知权限申请）
        │   ├── data/
        │   │   ├── model/                   # Song / LyricLine / Quality
        │   │   ├── remote/                  # KuwoApi / KuwoParser / NetworkModule
        │   │   └── repository/              # MusicRepository（四大接口）
        │   ├── player/
        │   │   ├── MusicPlayer.kt           # ExoPlayer 单例 + 状态流 + 进度轮询
        │   │   └── PlaybackService.kt       # Media3 后台播放与媒体通知
        │   └── ui/
        │       ├── MainViewModel.kt         # 搜索/分页/播放调度/歌词/音质
        │       ├── FengYinRoot.kt           # 导航壳 + 迷你播放条 + 播放页
        │       ├── components/MiniPlayer.kt
        │       ├── home/HomeScreen.kt
        │       ├── player/PlayerScreen.kt
        │       ├── profile/ProfileScreen.kt
        │       └── theme/Theme.kt
        └── res/
            ├── values/ (colors / strings / themes)
            ├── xml/network_security_config.xml   # 放行 kuwo.cn、nxinxz.com 明文流量
            ├── drawable/ic_launcher_foreground.xml
            └── mipmap-anydpi-v26/ic_launcher.xml
```

## 3. 技术栈

| 项 | 版本 |
| --- | --- |
| Kotlin / AGP / Gradle | 2.0.20 / 8.5.2 / 8.9 |
| compileSdk / targetSdk / minSdk | 34 / 34 / 26（Android 8.0+） |
| UI | Jetpack Compose（BOM 2024.09.00）+ Material 3 |
| 播放内核 | Media3 ExoPlayer 1.4.1 + Media3 Session（后台播放/通知） |
| 网络 | Retrofit 2.11 + OkHttp 4.12 + Gson |
| 图片 | Coil 2.7 |

## 4. 构建方式

```bash
# 方式一：Android Studio
#   File → Open → 选择 fengyin-android 目录，等待 Gradle Sync 后运行 app

# 方式二：命令行（需本机已装 JDK 17 与 Android SDK）
export ANDROID_HOME=/path/to/Android/Sdk
gradle assembleDebug          # 首次需联网下载依赖
# 产物：app/build/outputs/apk/debug/app-debug.apk
```

> 本机无 JDK / Gradle / Android SDK，交付物为完整源码工程，未做真机编译。

## 5. 已实现能力

- 关键词搜索 + 分页浏览（上一页 / 下一页 / 到底提示）
- 点击列表播放，列表首尾循环的上/下一首
- 播放页：旋转封面、逐行歌词高亮与自动滚动、点击歌词行跳转播放
- 进度条拖拽 seek，时间显示
- 标准 128K / 高品质 320K / 无损 FLAC 三档音质切换（保留播放进度）
- 底部迷你播放条（封面 + 标题 + 进度 + 播放/下一首），点击进播放页
- 后台播放与系统媒体通知（Media3 `PlaybackService`）
- 深浅色主题跟随系统

## 6. 已知限制

- 音频直链依赖第三方解析接口，接口可用性随对方策略变化，不保证长期稳定
- 未实现登录、收藏、歌单、本地下载与缓存
- 歌词为原文逐行（未做逐字卡拉 OK 效果）
- 搜索结果不落库，切页即覆盖

## 7. 免责声明

本项目仅用于技术学习与交流，音频内容版权归原平台及权利人所有，请勿用于任何商业用途。
