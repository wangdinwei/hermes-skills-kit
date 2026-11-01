# 对标 MV 形态实测：把「像哪种片子」量出来

用户说「去 B 站看看人家那种 MV」「哪种风格合适」时，不要凭印象回答。按这套跑一遍，能拿到播放量、均镜长、画面形态三组硬数据。

## 一、取候选

B 站的 search API 直接 curl 会被风控打回 HTML（`api.bilibili.com/x/web-interface/search/type` 返回整页 `<html>`），**用浏览器打开搜索页抓 DOM**：

```python
# browser_exec
goto_url("https://search.bilibili.com/all?keyword=" + quote("AI虚拟歌手 单曲 MV"))
wait_for_load()
rows = js("""(() => {const seen=new Set(), out=[];
 for (const a of document.querySelectorAll('a[href*=\"/video/BV\"]')) {
   const h=a.href.split('?')[0]; if(seen.has(h))continue; seen.add(h);
   const c=a.closest('.bili-video-card')||a.closest('div');
   const t=c&&c.querySelector('.bili-video-card__info--tit, h3');
   const au=c&&c.querySelector('.bili-video-card__info--author');
   const du=c&&c.querySelector('.bili-video-card__stats__duration');
   out.push({title:(t.textContent||'').trim(), url:h,
             author:au?au.textContent.trim():'', dur:du?du.textContent.trim():''});}
 return JSON.stringify(out.slice(0,30));})()""")
```

多组关键词各搜一遍（形态词 × 赛道词），按 URL 去重，剔掉时长超过 1 小时的教学/直播。

## 二、取元数据（yt-dlp 必须隔离 Python 环境）

本机 yt-dlp 会加载到别的 venv 的 `pycryptodome`，报 `Version mismatch: cffi 2.1.1 vs _cffi_backend 2.0.0` 直接崩。**所有调用都清掉这三个变量**：

```bash
env -u PYTHONPATH -u VIRTUAL_ENV -u PYTHONHOME \
  yt-dlp -J --skip-download "https://www.bilibili.com/video/$BV/"
```

`-J` 给 `view_count` / `like_count` / `duration` / `uploader` / `tags` / `description`，**播放量只在这里拿得到（搜索页 DOM 里没有）**。字段可能是 `None`，格式化时用 `r["view"] or 0`，别直接 `:>9`（会崩在 `NoneType.__format__`）。

## 三、量形态（这才是判据）

```bash
# 下 480p 样片（15MB 级，够判形态，不用 1080p）
env -u PYTHONPATH -u VIRTUAL_ENV -u PYTHONHOME \
  yt-dlp -q -f "bv*[height<=480]+ba/b[height<=480]/b" \
  --merge-output-format mp4 -o "vid/$name.mp4" "$url"

# 剪辑点：场景切换计数（-v quiet 会吞掉 showinfo，必须留 info 级）
ffmpeg -hide_banner -i in.mp4 -vf "select='gt(scene,0.3)',showinfo" -vsync 0 -f null - 2>&1 \
  | grep -c "pts_time:"
# 均镜长 = 时长 / 剪辑点

# 4 帧拼图：5% / 30% / 60% / 88% 各抽一帧，2×2 vstack(hstack)
ffmpeg -i f0.jpg -i f1.jpg -i f2.jpg -i f3.jpg -filter_complex \
 "[0:v]scale=480:-1[a];[1:v]scale=480:-1[b];[2:v]scale=480:-1[c];[3:v]scale=480:-1[d];\
  [a][b]hstack[t];[c][d]hstack[u];[t][u]vstack" 拼图.jpg
```

拼图交给 `vision_analyze`，一次问五件事：美术风格（实拍/AI 写实/插画/3D/文字排版）、有没有人露正脸唱、有没有烧硬字幕、场景与色调、一眼是否 AI。

## 四、判读口径（踩过的坑）

| 数据 | 能说明什么 | 不能说明什么 |
|---|---|---|
| 播放量高 | 那种形态在这个平台有人看 | **不能**说明用户要的就是它 |
| 均镜长 <3 秒 | 剪辑密度高（3D 舞台/战斗类） | 不等于更高级 |
| 均镜长 >30 秒、剪辑点个位数 | 文字排版/单图长镜，成本近零 | 这类常是播放最高的，但用户明确不要 |
| 五支里五支都烧硬字幕 | 字幕是这类片子的标配，不是可选 | — |
| 几乎都不对口型 | 氛围型的惯例 | **演唱型要求正好相反**，别把这条当通则 |

**硬结论：形态选择由用户要什么决定，数据只用来描述现状。** 把播放量当成路线依据，就会推荐「零角色、41 秒一镜」的形态，然后被否。

另一条实测：B 站这类片子基本是 16:9 横屏（852×480 下载），**投抖音必须重新裁 9:16，构图和字幕位置全部重排**，横屏片子不能直接搬。
