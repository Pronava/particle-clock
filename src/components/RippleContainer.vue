<template>
  <canvas ref="canvas" class="webgl-canvas"></canvas>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue";

const canvas = ref(null);
let gl = null;
let animationId = null;
let program = null;
let startTime = 0;

const vertexShaderSource = `
  attribute vec2 a_position;
  varying vec2 v_uv;

  void main() {
    v_uv = (a_position + 1.0) / 2.0; // 转到[0,1]区间UV
    gl_Position = vec4(a_position, 0.0, 1.0);
  }
`;

const fragmentShaderSource = `
  precision mediump float;

  uniform sampler2D u_texture;
  uniform float u_time;
  varying vec2 v_uv;

  void main() {
    // 水波折射参数
    float wave = 0.02;
    float speed = 3.0;
    float freq = 20.0;

    // 计算折射偏移量（波纹扰动UV坐标）
    float offsetX = wave * sin(freq * v_uv.y + speed * u_time);
    float offsetY = wave * cos(freq * v_uv.x + speed * u_time);

    vec2 uv = v_uv + vec2(offsetX, offsetY);

    // 从纹理采样颜色
    vec4 color = texture2D(u_texture, uv);

    gl_FragColor = color;
  }
`;

function createShader(gl, type, source) {
  const shader = gl.createShader(type);
  gl.shaderSource(shader, source);
  gl.compileShader(shader);
  if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
    console.error("Shader compile failed:", gl.getShaderInfoLog(shader));
    return null;
  }
  return shader;
}

function createProgram(gl, vertexShader, fragmentShader) {
  const program = gl.createProgram();
  gl.attachShader(program, vertexShader);
  gl.attachShader(program, fragmentShader);
  gl.linkProgram(program);
  if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
    console.error("Program link failed:", gl.getProgramInfoLog(program));
    return null;
  }
  return program;
}

function loadTexture(gl, image) {
  const texture = gl.createTexture();
  gl.bindTexture(gl.TEXTURE_2D, texture);

  // 设置参数防止纹理大小限制
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);

  // 上传图片数据
  gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, image);
  return texture;
}

function initGL(image) {
  const c = canvas.value;
  const dpr = window.devicePixelRatio || 1;

  // 设置画布尺寸为CSS尺寸×dpr，保证高清
  c.width = c.clientWidth * dpr;
  c.height = c.clientHeight * dpr;

  gl = c.getContext("webgl") || c.getContext("experimental-webgl");
  if (!gl) {
    alert("浏览器不支持WebGL");
    return;
  }

  gl.viewport(0, 0, c.width, c.height);

  const vertexShader = createShader(gl, gl.VERTEX_SHADER, vertexShaderSource);
  const fragmentShader = createShader(gl, gl.FRAGMENT_SHADER, fragmentShaderSource);
  program = createProgram(gl, vertexShader, fragmentShader);

  gl.useProgram(program);

  // 顶点数据，绘制2个三角形覆盖整个画布
  const positionBuffer = gl.createBuffer();
  gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
  const positions = new Float32Array([
    -1, -1, 1, -1, -1, 1,
    -1, 1, 1, -1, 1, 1,
  ]);
  gl.bufferData(gl.ARRAY_BUFFER, positions, gl.STATIC_DRAW);

  const aPositionLoc = gl.getAttribLocation(program, "a_position");
  gl.enableVertexAttribArray(aPositionLoc);
  gl.vertexAttribPointer(aPositionLoc, 2, gl.FLOAT, false, 0, 0);

  // 创建纹理
  const texture = loadTexture(gl, image);

  // 绑定纹理到uniform
  const uTextureLoc = gl.getUniformLocation(program, "u_texture");
  gl.uniform1i(uTextureLoc, 0); // 纹理单元0

  startTime = performance.now();

  function render() {
    const currentTime = (performance.now() - startTime) / 1000;

    // 传时间
    const uTimeLoc = gl.getUniformLocation(program, "u_time");
    gl.uniform1f(uTimeLoc, currentTime);

    // 清屏
    gl.clearColor(0, 0, 0, 1);
    gl.clear(gl.COLOR_BUFFER_BIT);

    // 绑定纹理
    gl.activeTexture(gl.TEXTURE0);
    gl.bindTexture(gl.TEXTURE_2D, texture);

    // 绘制
    gl.drawArrays(gl.TRIANGLES, 0, 6);

    animationId = requestAnimationFrame(render);
  }
  animationId = requestAnimationFrame(render);
}

onMounted(() => {
  const img = new Image();
  img.crossOrigin = "anonymous";
  // 这里换成你自己的图片地址或本地静态资源路径
  img.src = "https://images.unsplash.com/photo-1506744038136-46273834b3fb?auto=format&fit=crop&w=600&q=80";

  img.onload = () => {
    initGL(img);
  };
});

onBeforeUnmount(() => {
  if (animationId) cancelAnimationFrame(animationId);
});
</script>

<style scoped>
.webgl-canvas {
  width: 600px;
  height: 400px;
  border: 1px solid #333;
  display: block;
  image-rendering: pixelated;
}
</style>
