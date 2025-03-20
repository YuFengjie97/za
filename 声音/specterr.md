## https://app.specterr.com/create 

### createPage
createPage.jsx通过查看路由得知
关键: import AudioElement from './components/PlaybackControls/AudioElement';

### AudioElement
封装的audio节点, 生命周期将element节点传入`RealtimeAudioAnalyzer`
通过state传入该组件props,来初始化RealtimeAudioAnalyzer的一些参数
```
audioUrl,
audioMuted,
bassMaxMagnitude,
wideMaxMagnitude
```

### RealtimeAudioAnalyzer
这里面有个window全局的上下文,在哪里初始化,未知`window.realtimeAnalyzerContext`
封装了很多方法直接获取realtimeAnalyzerContext的参数或对其设置
这个上下的参数初始化应该是来自`AudioAnalyzerOptions`
其中还有很多方法是直接设置这个上下文,应该是在运行时/用户输入来改变一些参数
其中关键的有这两个方法,分别是获取低频和宽频的频率数据`getCurrentBassFrequency` `getCurrentWideFrequency`
另外还看到了historyFrequency的字样,应该是根据历史频率数据来设置当前频率,保证数据的连续性

### AudioAnalyzerOptions
```js

export const AudioAnalyzerOptions = {
    fftSize: 1024 * 8, // fftsize看上去是固定写死的,运行时并没有对其进行过修改
    smoothingTime: 0.1,
    
    minDecibels: -100,
    maxDecibels: -0,

    // The max length of bins we need for processing.
    clearFrequencyMaxLength: 1024
};


export const BassSpectrumOptions = {
    spectrumStart: 3,
    spectrumEnd: 26,

    exponent: 10,
    root: 1,
    magnitudeTargetMax: 32,
    baseBarCount: 200
};


export const WideSpectrumOptions = {
    spectrumStart: 0,
    spectrumEnd: 225,

    exponent: 5,
    root: 2,
    magnitudeTargetMax: 150,
    baseBarCount: 200
};

```


### getCurrentBassFrequency
获取低频频率数据
```js


const getCurrentBassFrequency = () => {
  // 判断上下文不存在直接返回空
    if (!window.realtimeAnalyzerContext || !window.realtimeAnalyzerContext.analyzerNode) {
        return EmptyAudioFrequency;
    }
        
  // 重新创建Float32Array来获取波形数据,这里为什么需要波形数据呢?
  // 而且array的length应该是bufferLength,应该是fftSize的一半,这里是否是作者的bug?
    let timeDomainData = new Float32Array(window.realtimeAnalyzerContext.analyzerNode.fftSize);
    window.realtimeAnalyzerContext.analyzerNode.getFloatTimeDomainData(timeDomainData);


    // 看上去是对频率数据的截取,clearFrequencyMaxLength的长度是固定的1024,
    let clearFrequency = window.realtimeAnalyzerContext.getByteFrequencyData(timeDomainData, AudioAnalyzerOptions.clearFrequencyMaxLength);

// 根据设置中低频频桶的位置范围进行获取具体的低频频率桶
    // Get the bass range only.
    let bassFrequency = clearFrequency.slice(BassSpectrumOptions.spectrumStart, BassSpectrumOptions.spectrumEnd);


// 对获取到的低频频率桶的数据进一步修饰
    bassFrequency = decorateAudioFrequency(
        bassFrequency, // 获取到的低频频率桶, float32Array
        window.realtimeAnalyzerContext.bassTargetMax, // 低频的目标值????
        window.realtimeAnalyzerContext.bassExponent, // 低频指数,猜测是用来加强低频的指数值
        window.realtimeAnalyzerContext.bassRoot, // 低频根,猜测是用来缩小差异的根值
        window.realtimeAnalyzerContext.bassBaseBarCount, // 最后转换后得到的低频数据的长度
        window.realtimeAnalyzerContext.bassMaxMagnitude ? window.realtimeAnalyzerContext.bassMaxMagnitude : DEFAULT_AUDIO_TRACK.BASS_MAX_MAGNITUDE // 低频的最大振幅,做了三元运算,那就说明在运行时可能修改过低频最大振幅值
    );
    
    return bassFrequency;
};

```

### bassMaxMagnitude,低音的频率最大振幅值
猜测可能在播放之前,歌曲载入之后,计算得到了该值

### MaxMagnitudeAnalyzer.js 最大振幅值分析
`analyzeCumulativeMaxMagnitudes`应该是这里对一段事件的频率的最大振幅进行了计算
```js
/**
 * Analyze the audio file and calculate the cumulative max magnitudes for bass and wide frequencies.
 * The function takes an audio file or url as an argument and returns an object with properties bassMaxMagnitude and wideMaxMagnitude.
 * The cumulative max magnitudes are calculated by taking the top X percent of the max values from the audio analysis.
 */
/**
 * 分析音频文件并计算低音和宽频的累积最大幅度。
 * 该函数将音频文件或 url 作为参数，并返回一个具有 bassMaxMagnitude 和 wideMaxMagnitude 属性的对象。
 * 累计最大幅度的计算方法是取音频分析中最大值的前 X%。
 */
export async function analyzeCumulativeMaxMagnitudes(audioFile, onProgress, abortSignal) {

    abortSignal?.addEventListener('abort', () => { throw new Error('Analysis aborted') }, { once: true });

    let audioAnalysisProgress = {
        [AUDIO_ANALYSIS_PHASES.PREPARING]: 0,
        [AUDIO_ANALYSIS_PHASES.DECODING]: 0,
        [AUDIO_ANALYSIS_PHASES.ANALYZING]: 0
    }

    const updateAnalysisProgress = (key, value) => {
        audioAnalysisProgress[key] = value;
        const progress = calculateAverageProgress(audioAnalysisProgress);
        onProgress(progress);
    }

    // Set the default values for the audio analysis.
    const magnitudes = {
        bassMaxMagnitude: AudioAnalyzerConstants.DefaultBassMaxMagnitude,
        wideMaxMagnitude: AudioAnalyzerConstants.DefaultWideMaxMagnitude
    };

    const getArrayBufferFunc = audioFile instanceof File ? getArrayBufferFromFile : getArrayBufferFromUrl;
    let arrayBuffer = await getArrayBufferFunc(audioFile, (progress) => updateAnalysisProgress(AUDIO_ANALYSIS_PHASES.PREPARING, progress), abortSignal);
    let audioContext = new (window.AudioContext || window.webkitAudioContext)();
    let audioBuffer = await decodeAudioWithProgress(audioContext, arrayBuffer, (progress) => updateAnalysisProgress(AUDIO_ANALYSIS_PHASES.DECODING, progress));

    console.log('Audio info for analysis:', audioBuffer);

    const durationInSeconds = audioBuffer.duration;
    const sampleRate = audioBuffer.sampleRate;

    // Mix the audio channels.
    // Need to split this by chunks if this operation is expensive for browser's work.
    let mixedSamples = new Float32Array(audioBuffer.length);
    for (let i = 0; i < audioBuffer.numberOfChannels; i++) {
        let channelSamples = audioBuffer.getChannelData(i);
        for (let j = 0; j < channelSamples.length; j++) {
            mixedSamples[j] += channelSamples[j] / audioBuffer.numberOfChannels;
        }
    }

    const audioAnalysisFps = AudioAnalyzerConstants.MaxMagnitudeAnalysisFps;
    const stepInSeconds = 1 / audioAnalysisFps;
    let previousPercentage = 0;
    const percentageStep = 3;

    // Create the custom fft processor for audio analysis.
    const getByteFrequencyData = createFFTAnalyzer(
        AudioAnalyzerOptions.fftSize,
        AudioAnalyzerOptions.minDecibels,
        AudioAnalyzerOptions.maxDecibels,
        AudioAnalyzerOptions.smoothingTime
    );

    let bassMaxMagnitudes = [];
    let wideMaxMagnitudes = [];

    const clearAudioResources = () => {
        audioBuffer = null;
        arrayBuffer = null;
        mixedSamples = null;
    
        if (audioContext) {
            audioContext.close();
            audioContext = null;
        }
    };

    // Iterate through the audio file and calculate the bass and wide max magnitudes.
    for (let i = 0; i < durationInSeconds; i += stepInSeconds) {
        if (abortSignal.aborted) {
            clearAudioResources();
            throw new Error('Analysis aborted');
        }

        const startSample = Math.floor(i * sampleRate);
        const endSample = Math.min(startSample + AudioAnalyzerOptions.fftSize, audioBuffer.length);
        let timeDomainData = mixedSamples.subarray(startSample, endSample);

        // Convert the time domain data to frequency domain data(0-255 values).
        let clearFrequency = getByteFrequencyData(timeDomainData, AudioAnalyzerOptions.clearFrequencyMaxLength);

        // Calculate the bass max magnitude.
        let bassFrequency = clearFrequency.subarray(BassSpectrumOptions.spectrumStart, BassSpectrumOptions.spectrumEnd);
        let bassFrequencyMax = Math.max(...bassFrequency);

        // Calculate the wide max magnitude.
        let wideFrequency = clearFrequency.subarray(WideSpectrumOptions.spectrumStart, WideSpectrumOptions.spectrumEnd);
        let wideFrequencyMax = Math.max(...wideFrequency);

        bassMaxMagnitudes.push(bassFrequencyMax);
        wideMaxMagnitudes.push(wideFrequencyMax);

        let percentage = (i / durationInSeconds) * 100;
        if (percentage > previousPercentage + percentageStep || percentage >= 100) {
            previousPercentage = percentage;
            updateAnalysisProgress(AUDIO_ANALYSIS_PHASES.ANALYZING, percentage);

            // Wait for 0ms to let the UI update.
            // Without this the UI will be blocked to the end of audio analysis.
            await new Promise((resolve, reject) => { setTimeout(resolve, 0); });
        }
    }

    // Calculate the cumulative max magnitudes.
    let percentile = AudioAnalyzerConstants.MaxMagnitudeCalculationPercentile;
    magnitudes.bassMaxMagnitude = calculateCumulativeMaxMagnitude(bassMaxMagnitudes, percentile);
    magnitudes.wideMaxMagnitude = calculateCumulativeMaxMagnitude(wideMaxMagnitudes, percentile);

    clearAudioResources();
    return magnitudes;
}
```
这个计算低频宽频最大振幅-->store里analyzeAudioMaxMagnitudes,存在store里,参数是auidoUrl,是选择歌曲后,对歌曲进行了计算,有看到ui组件中的click触发了这个action

### getCurrentWideFrequency
跟低频类似

### decorateAudioFrequency 对频率数据的自定义整理
```js

// Decorates the audio frequency to receive the final values for bass or wide.
export const decorateAudioFrequency = (spectrumFrequency, magnitudeTargetMax, exponent, root, baseBarCount, cumulativeMaxMagnitude) => {
    const targetMax = magnitudeTargetMax;

    // Step 1.
    // Normalizes all the values to maximum value.
    // Check the extra high bins and reduce them to get accurate result.
// 步骤 1.
    // 将所有值归一化为最大值。
    // 检查额外的高分带并将其减少，以获得准确的结果。
    let magnitudeMultiplier = targetMax / cumulativeMaxMagnitude;
    let normalizedBin = 0;

    for (let i = 0; i < spectrumFrequency.length; i++) { 
        normalizedBin = spectrumFrequency[i] * magnitudeMultiplier;
        spectrumFrequency[i] = normalizedBin <= targetMax ? normalizedBin : targetMax;
    }

    // Step 2.
    // Exaggerate the peaks of a data array with a target max. All values beneath the target max are reduced.
    // The further below the target max a value is, the more it is reduced. Values at or near the target max are barely changed.
  // 步骤 2.
    // 用目标最大值夸大数据数组的峰值。所有低于目标最大值的值都会减小。
    // 目标最大值越低，减小的幅度越大。处于或接近目标最大值的值几乎不会发生变化。
    if (exponent > 1) {
        let divisor = Math.pow(targetMax, exponent - 1);
        for (var i = 0; i < spectrumFrequency.length; i++) {
            spectrumFrequency[i] = Math.pow(spectrumFrequency[i], exponent) / divisor;
        }
    }

    // Step 3.
    // Does the opposite of exaggerate.
    // The further below the target max a value is, the more it is increased, approaching the target max.
    // This reduces the exaggeration of peaks in the array. 

    // 第 3 步
    // 与夸大相反。
    // 一个值越低于目标最大值，它的增大幅度就越大，越接近目标最大值。
    // 这样可以减少数组中峰值的夸大。
    if (root > 1) {
        let exponent = 1 / root;
        let multiplier = Math.pow(Math.pow(targetMax, exponent), root - 1);
    
        for (var i = 0; i < spectrumFrequency.length; i++) {
            spectrumFrequency[i] = Math.pow(spectrumFrequency[i], exponent) * multiplier;
        }
    }

    // Return the final result.
    return conformToBarCount(spectrumFrequency, baseBarCount);
};

```

### PreviewDisplay.jsx
因为最后导出的是视频,右侧预览是pixi绘制canvas实时预览的,猜测,就是在这里接受频率数据进行绘制的.而且我只查到了`getCurrentBassFrequency`的使用,全局下没有看到对`getCurrentWideFrequency`,猜测,只使用了低频特征来表现动画节奏.另外低频一般是鼓点的所在区域,的确是适用于节奏的绘制