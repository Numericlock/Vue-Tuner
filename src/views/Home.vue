<template>
    <div>
        <h1>AbsScale<br>click to start</h1>
        <canvas id="canvas" ref="canvas" width="640" height="300"></canvas>
    </div>
</template>

<script>
    export default {
        components: {},
        props: {},
        data: function() {
            return {
                audio:null,
                analyser:null,
                source:null,
            };
        },
        methods: {
            init:async function(fftsize) {
                this.audio = new(window.AudioContext || window.webkitAudioContext)()
                const stream = await navigator.mediaDevices.getUserMedia({
                    audio: true
                })
                this.source = this.audio.createMediaStreamSource(stream)
                this.analyser = this.audio.createAnalyser() // FFT
                this.analyser.minDecibels = -90
                this.analyser.maxDecibels = -10
                this.analyser.smoothingTimeConstant = 0 // 0-.999 0.85 ふわっとなる
                this.analyser.fftSize = 1 << fftsize // min:32==1<<5 max:32768==1<<15
                this.source.connect(this.analyser)
                //this.analyser = analyser

                console.log("audio.sampleRate: ", this.audio.sampleRate) // 44100 固定?
                this.resolution = this.audio.sampleRate / this.analyser.fftSize
                console.log("resolution: ", this.resolution + "Hz") // 分解能

                const buflen = this.analyser.frequencyBinCount
                this.buf = new Uint8Array(buflen)
            },
            getFreqs() {
                this.analyser.getByteFrequencyData(this.buf)
                return this.buf
            },
            getScale() {
                this.analyser.getByteFrequencyData(this.buf)
                const formant = this.getFormant(this.buf)
                if (formant.length == 0)
                    return null
                /*
    let s = "" + formant.length + ": "
    for (let i = 0; i < Math.min(formant.length, 5); i++) {
      const f = formant[i]
//      s += f[0] + "(" + f[1] + ") "
      s += f[0] + " "
    }
    if (this.bks != s) {
      //console.log(s)
      this.bks = s
    }
    */
                return this.getScaleFromFreq(formant[0][0])
            },
            getFormant(buf) {
                const thr = 50
                const formant = []

                let state = 0 // 0:<thr, 1:increse, 2:decrese (push formant when 1 -> 2)
                let bkn = 0
                for (let i = 0; i < buf.length; i++) {
                    const n = buf[i]
                    if (state == 0) {
                        if (n >= thr) {
                            state = 1
                        }
                    } else if (state == 1) {
                        if (n < bkn) {
                            formant.push([i - 1, bkn])
                            if (n < thr)
                                state = 0
                            else
                                state = 2
                        }
                    } else if (state == 2) {
                        if (n > bkn) {
                            state = 1
                        } else if (n < thr)
                            state = 0
                    }
                    bkn = n
                }
                formant.sort((a, b) => b[1] - a[1])
                for (let i = 0; i < formant.length; i++) {
                    formant[i][0] *= this.resolution
                }
                return formant
            },
            getsScaleFromFreq(freq, detail) {
                console.log(freq);
                let d = 12 * Math.log2(freq / 440) % 12
                if (d <= -0.5)
                    d += 12
                const d1 = Math.round(d)

                const d2 = d - d1
                //console.log(freq, d2, d1, d2)
                let scale = ["ラ", "ラ#", "シ", "ド", "ド#", "レ", "レ#", "ミ", "ファ", "ファ#", "ソ", "ソ#", "ラ"][d1]
                if (detail) scale += " " + (d2 >= 0 ? "+" : "") + d2.toFixed(1)
                return scale
            },
            getScaleFromFreq(freq, detail) {
                //console.log(freq);
                let d = 12 * Math.log2(freq / 440) % 12
                if (d <= -0.5)
                    d += 12
                const d1 = Math.round(d)

                const d2 = d - d1
                //console.log(freq, d2, d1, d2)
                let scale = ["A", "A#", "B", "C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A"][d1]
                if (detail) scale += " " + (d2 >= 0 ? "+" : "") + d2.toFixed(1)
                return scale
            },
            getOctave(freq) {
                let result;
                const octave = ['lowlowlow','lowlow','low','mid1','mid2','hi','hihi','hihihi','hihihihi'];
                if(freq >= 440){
                    for(let i = 0;i < 5;i++){
                        if(freq < 440 * ((i + 1)* 2)){
                            result = 5+i;
                            break;
                        }
                    }
                }else{
                    for(let i = 0;i < 5;i++){
                        if(freq >= 440 * (1/ (2*(i+1)))){
                            result = 4-i;
                            break;
                        }
                    }
                }
                let resultOctave = octave[result];
                if(resultOctave === undefined) resultOctave = 'noise';
                console.log(resultOctave);
                return resultOctave;
            }
        },
        mounted() {
            let fftsize = 15
            this.init(fftsize)
            var canvas = this.$refs.canvas;
            const g = canvas.getContext('2d')
            let buf = this.getFreqs()

            let bks = ""
            let bkf = ""
            const draw = function() {
                canvas.width = canvas.clientWidth
                canvas.height = canvas.clientHeight
                const cw = canvas.width
                const ch = canvas.height
                const bw = cw / buf.length * 10

                buf = this.getFreqs()
                g.fillStyle = 'rgb(0, 0, 0)'
                g.fillRect(0, 0, cw, ch)
                let x = 0
                for (let i = 0; i < buf.length; i++) {
                    const n = buf[i]
                    g.fillStyle = 'hsl(' + n + ',100%,50%)'
                    g.fillRect(x, ch - n, bw, n)
                    x += bw
                }
                const formant = this.getFormant(buf)
                if (formant.length > 0) {
                    const freq = formant[0][0]
                    const scale = this.getScaleFromFreq(freq)
                    const scale2 = this.getScaleFromFreq(freq, true)
                    bks = this.getOctave(freq) + ":" + scale
                    bkf = Math.floor(freq + .5) + "Hz " + scale2
                }
                //console.log(scale)
                g.textAlign = "center"
                g.fillStyle = "#fff"
                g.font = "20vw sans-serif"
                g.textBaseline = "bottom"
                g.fillText(bks, cw / 2, ch / 2)
                g.font = "5vw sans-serif"
                g.textBaseline = "top"
                g.fillText(bkf, cw / 2, ch / 2)

                requestAnimationFrame(draw)
            }
            draw()
        }
    }
</script>

<style lang="scss">
    body {
        margin: 0;
        background-color: black;
    }

    #canvas {
        position: absolute;
        width: 100vw;
        height: 100vh;
    }

    h1 {
        position: absolute;
        width: 100vw;
        text-align: center;
        padding: 30vh 0;
        color: white;
    }
</style>