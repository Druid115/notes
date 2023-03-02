# 连续图片转视频
## ffmpeg
- 依赖：需要安装ffmpeg客户端
- 优点：效率很高，命令简单，支持之间转出264编码的mp4，可以在Google Chrome 播放
- 缺点：现在命令转化对图片的命名有严格要求，比如%d.png 就是0.png开始 1、2、3中间不能断
- 命令示例(%d.png 表示文件名格式，libx264指定编码，tt2.mp4 表示输出视频文件位置)
```cmd
//转化为264格式
ffmpeg -f image2 -i %d.png  -vcodec libx264 -s 1920*1080 -r 10  test.mp4
//转化为flv格式
ffmpeg -f image2 -i %d.png -r 10 -f flv test.flv
```

## JAVA 代码
- 依赖：需要使用第三方jar包 Jim2mov.jar customizer.jar jmf.jar mediaplayer.jar multiplayer.jar
- 优点：代码控制，拓展性较强。
- 缺点：转出MP4格式不为H264编码。使用专业播放器可以播放，但是不支持H5标签直接播放。
- 示例代码：
```java
package com.youngth.zeus.img2video;

import java.io.File;
import java.util.ArrayList;

import org.jim2mov.core.DefaultMovieInfoProvider;
import org.jim2mov.core.FrameSavedListener;
import org.jim2mov.core.ImageProvider;
import org.jim2mov.core.Jim2Mov;
import org.jim2mov.core.MovieInfoProvider;
import org.jim2mov.core.MovieSaveException;
import org.jim2mov.utils.MovieUtils;

public class FilesToMov implements ImageProvider, FrameSavedListener{
	// 文件数组
    private ArrayList<String> fileArray = null;
    // 文件类型
    private int type = MovieInfoProvider.TYPE_QUICKTIME_JPEG;
    // 主函数
	public static void main(String[] args) throws MovieSaveException {
		ArrayList<String> fileArray = new ArrayList<>();
		File[] listFiles = new File("D:\\zhouys\\self\\img2video\\src\\main\\resources\\img").listFiles();
		
		for (int i = 0; i < listFiles.length; i++) {
			fileArray.add(listFiles[i].getAbsolutePath());
		}
		new FilesToMov(fileArray, MovieInfoProvider.TYPE_QUICKTIME_JPEG, "Test1.mp4");
		new FilesToMov(fileArray, MovieInfoProvider.TYPE_AVI_MJPEG, "Test2.mp4");
		new FilesToMov(fileArray, MovieInfoProvider.TYPE_AVI_RAW, "Test3.mp4");
	}
	
	/**
	 * 图片转视频
	 * @param fileArray 文件路径数组
	 * @param type 格式
	 * @param path 文件名
	 * @throws MovieSaveException 
	 */
	public FilesToMov(ArrayList<String> fileArray, int type, String path) throws MovieSaveException {
		this.fileArray = fileArray;
		this.type = type;
		DefaultMovieInfoProvider dmip = new DefaultMovieInfoProvider(path);
		// 设置帧频率
		dmip.setFPS(1);
		// 设置帧数--一张图片一帧
		dmip.setNumberOfFrames(fileArray.size());
		// 设置视频高度
		dmip.setMWidth(320);
		// 设置视频宽度
		dmip.setMHeight(240);
		new Jim2Mov(this, dmip, this).saveMovie(this.type);;
	}

	@Override
	public void frameSaved(int frameNumber) {
        System.out.println("Saved frame: " + frameNumber);
	}

	@Override
	public byte[] getImage(int frame) {
		try {
			return MovieUtils.convertImageToJPEG(new File(fileArray.get(frame)), 1.0f);
		} catch (Exception e) {
			e.printStackTrace();
		}
		return null;
	}
}
```
# 视频转流
- 直接将本地视频推出rtmp流
```cmd
ffmpeg -re -stream_loop -1  -i test.mp4 -c copy -f flv rtmp://192.168.100.63:1935/myapp/testzys
```
# 图片直接转流
- 优点：不用转存视频文件
- 缺点：无法动态添加图片
```cmd
ffmpeg -re -stream_loop -1 -f image2 -i %d.png -r 12 -s 1920*1080 -f flv rtmp://192.168.100.63:1935/myapp/testzys1
```
# 使用javacv实现 图片到rtmp流
- 基于opencv和ffmpeg，但是在顶层代码上做了封装
- 优点：结合了代码操作和ffmpeg的优点，解决了Jim2mov的问题,可以动态动图读取
- 缺点：（待总结）
- 依赖
```pom
        <!-- https://mvnrepository.com/artifact/org.bytedeco/javacv -->
        <dependency>
            <groupId>org.bytedeco</groupId>
            <artifactId>javacv</artifactId>
            <version>1.5.2</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.bytedeco/javacv-platform -->
        <dependency>
            <groupId>org.bytedeco</groupId>
            <artifactId>javacv-platform</artifactId>
            <version>1.5.2</version>
        </dependency>
```
- 示例：
```java
package com.youngth.zeus.img2video;

import org.bytedeco.ffmpeg.global.avcodec;
import org.bytedeco.javacv.CanvasFrame;
import org.bytedeco.javacv.Frame;
import org.bytedeco.javacv.FrameRecorder;
import org.bytedeco.javacv.Java2DFrameConverter;

import javax.imageio.ImageIO;
import java.awt.*;
import java.awt.image.BufferedImage;
import java.io.FileInputStream;

/**
 * @author YoungTH Zeus
 * @date 2019/12/11
 */
public class JavacTest {
    public static void main(String[] args) {
// captureScreen();
        try {
            transferPicToRtmp("rtmp://192.168.100.63:1935/myapp/testzys1");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 图片转RTMP流
     *
     * @param outRtmpUrl
     * @throws Exception
     * @throws org.bytedeco.javacv.FrameRecorder.Exception
     * @throws InterruptedException
     */
    public static void transferPicToRtmp(String outRtmpUrl) throws Exception, org.bytedeco.javacv.FrameRecorder.Exception, InterruptedException {

        int frameRate = 25;
        FrameRecorder recorder;
        try {
            recorder = FrameRecorder.createDefault(outRtmpUrl, 800, 600);
        } catch (org.bytedeco.javacv.FrameRecorder.Exception e) {
            throw e;
        }

        recorder.setVideoCodec(avcodec.AV_CODEC_ID_H264);
        recorder.setFormat("flv");
        recorder.setFrameRate(frameRate);
        recorder.setGopSize(frameRate);

        System.out.println("准备开始推流...");
        try {
            recorder.start();
        } catch (org.bytedeco.javacv.FrameRecorder.Exception e) {
            try {
                System.out.println("录制器启动失败，正在重新启动...");
                if (recorder != null) {
                    System.out.println("尝试关闭录制器");
                    recorder.stop();
                    System.out.println("尝试重新开启录制器");
                    recorder.start();
                }
            } catch (org.bytedeco.javacv.FrameRecorder.Exception e1) {
                throw e;
            }
        }
        long startTime = 0;
        Frame capturedFrame = null;
        System.out.println("开始推流");
        int idx = 0;
        int temp = 0;
        Java2DFrameConverter co = new Java2DFrameConverter();
        String dir = "D:\\zhouys\\img2video\\image\\visible3\\";

        while (idx <= 904) {

            if (idx == 904) {
                idx = 0;
            }
            BufferedImage image = ImageIO.read(new FileInputStream(dir + idx + ".png"));
            capturedFrame = co.getFrame(image);
            if (startTime == 0) {
                startTime = System.currentTimeMillis();
            }
            long videoTs = 1000 * (System.currentTimeMillis() - startTime);
            if (videoTs > recorder.getTimestamp()) {
                recorder.setTimestamp(videoTs);// 时间戳
            }
            if (capturedFrame != null) {
                recorder.record(capturedFrame);
            }
            idx++;
            Thread.sleep(10);
            System.out.println(idx);
        }
        recorder.stop();
        recorder.release();
    }
}
```