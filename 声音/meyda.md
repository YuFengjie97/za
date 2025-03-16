|Feature|	含义|	用途示例|
|----|----|----|
|rms|	Root Mean Square（平均能量）衡量声音强度	|常用于做“节奏感跳动动画”|
|zcr|	Zero Crossing Rate（过零率）信号变化频率|	区分打击声/噪音/语音|
|spectralCentroid|	频谱质心（声音“亮度”）频谱重心位置|	区分明亮/低沉音色|
|spectralFlatness|	频谱平坦度衡量是否“嘈杂”/有旋律|	区分噪声和音符型声音|
|spectralRolloff|	频谱滚降点频率能量的90%落点|	可用作“音调高低”视觉映射|
|spectralSpread|	频谱离散程度|	感知声音是否集中或混杂|
|spectralSkewness|	频谱偏斜度|	分析频率倾斜方向|
|spectralKurtosis|	频谱峰度|	衡量频谱是否尖锐|
|amplitudeSpectrum|	原始频谱值数组|	可用于自定义频谱动画|
|powerSpectrum|	功率谱|	能量密度可视化|
|mfcc|	Mel-Frequency Cepstral Coefficients梅尔倒谱系数（听感特征）|	常用于语音识别 / 音乐情绪分析|
|chroma|	色度向量（12个音阶强度）|	音高感知/和弦分析|
|loudness|	音响响度（基于频带）|	模拟人耳响度响应|
|perceptualSpread|	感知频带扩散度|	声音空间感|
|perceptualSharpness|	感知尖锐度|	模拟人耳感知清晰度|
|energy|	声音能量（近似于 rms）|	可用于节奏同步|
|complexSpectrum|	复数频谱|	一般用于工程分析|
|buffer|	原始 PCM 数据|	自行处理原始数据|


|项目|	推荐特征组合|
|----|----|
|🎶 音乐节奏动画|	rms + spectralCentroid|
|🎧 可视化播放器|	amplitudeSpectrum + chroma|
|🤖 音频分类/识别|	mfcc + chroma + zcr|
|🎨 情绪视觉设计|	perceptualSharpness + loudness|