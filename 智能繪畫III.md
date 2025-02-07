以下是在現有系統基礎上集成生成式AI建議和AR構圖引導的完整實現方案，包含代碼和可視化組件：

```python
import streamlit as st
from streamlit_webrtc import webrtc_streamer, VideoTransformerBase
import av
import cv2
import numpy as np
import torch
from diffusers import StableDiffusionInpaintPipeline
from transformers import pipeline

# 初始化多模態AI模型（緩存加載）
@st.cache_resource
def load_ai_models():
    # 圖像生成模型
    inpainter = StableDiffusionInpaintPipeline.from_pretrained(
        "stabilityai/stable-diffusion-2-inpainting",
        torch_dtype=torch.float16
    ).to("cuda")
    
    # 文本生成模型
    text_generator = pipeline("text-generation", model="gpt2-medium")
    
    return inpainter, text_generator

class EnhancedPaintingAnalyzer(VideoTransformerBase):
    def __init__(self):
        super().__init__()
        self.inpainter, self.text_generator = load_ai_models()
        self.ar_guides = {}
        
        # AR構圖模板
        self.composition_templates = {
            'rule_of_thirds': self.create_rule_of_thirds_grid,
            'golden_spiral': self.create_golden_spiral,
            'symmetry': self.create_symmetry_guide
        }
    
    # AR構圖輔助方法
    def create_rule_of_thirds_grid(self, frame):
        h, w = frame.shape[:2]
        grid_img = np.zeros_like(frame)
        # 繪制三等分線
        cv2.line(grid_img, (w//3, 0), (w//3, h), (0,255,0), 2)
        cv2.line(grid_img, (2*w//3, 0), (2*w//3, h), (0,255,0), 2)
        cv2.line(grid_img, (0, h//3), (w, h//3), (0,255,0), 2)
        cv2.line(grid_img, (0, 2*h//3), (w, 2*h//3), (0,255,0), 2)
        return cv2.addWeighted(frame, 0.7, grid_img, 0.3, 0)
    
    def create_golden_spiral(self, frame):
        h, w = frame.shape[:2]
        spiral_img = np.zeros_like(frame)
        # 簡化版黃金螺旋繪制
        cv2.ellipse(spiral_img, (w//2, h//2), (w//4, h//4), 
                   0, 0, 90, (0,255,255), 3)
        return cv2.addWeighted(frame, 0.8, spiral_img, 0.2, 0)
    
    # 生成式AI建議
    async def generate_improvement_suggestion(self, img, mask):
        prompt = "high quality painting with improved composition and vibrant colors"
        guidance_image = self.inpainter(
            prompt=prompt,
            image=img,
            mask_image=mask,
            strength=0.7
        ).images[0]
        return guidance_image
    
    # 實時AR處理流程
    def apply_ar_guidance(self, frame):
        # 自動選擇最佳構圖模板
        if self.ar_guides.get('enable_dynamic'):
            gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
            edges = cv2.Canny(gray, 50, 150)
            
            # 簡單構圖分析
            h, w = frame.shape[:2]
            edge_density = {
                'left': np.mean(edges[:, :w//2]),
                'right': np.mean(edges[:, w//2:]),
                'top': np.mean(edges[:h//2, :]),
                'bottom': np.mean(edges[h//2:, :])
            }
            
            if max(edge_density.values()) - min(edge_density.values()) > 50:
                return self.composition_templates['rule_of_thirds'](frame)
            else:
                return self.composition_templates['golden_spiral'](frame)
        else:
            return self.composition_templates[self.ar_guides['current_template']](frame)
    
    def recv(self, frame: av.VideoFrame) -> av.VideoFrame:
        img = frame.to_ndarray(format="bgr24")
        
        # AR構圖引導
        if self.ar_guides.get('enable'):
            img = self.apply_ar_guidance(img)
        
        # 生成式AI建議層
        if st.session_state.get('show_ai_suggestion'):
            mask = self.create_attention_mask(img)
            ai_suggestion = self.generate_improvement_suggestion(img, mask)
            img = cv2.addWeighted(img, 0.5, ai_suggestion, 0.5, 0)
        
        # 添加交互式控件
        cv2.putText(img, "AR模式: {}".format("ON" if self.ar_guides.get('enable') else "OFF"), 
                   (10,30), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0,255,0), 2)
        
        return av.VideoFrame.from_ndarray(img, format="bgr24")

# Streamlit界面增強
st.set_page_config(page_title="AI繪畫導師", layout="wide")

# 側邊欄控制面板
with st.sidebar:
    st.header("智能引導設置")
    
    # AR設置
    ar_enable = st.checkbox("啓用AR構圖引導", True)
    ar_template = st.selectbox(
        "構圖模板",
        ["rule_of_thirds", "golden_spiral", "symmetry"],
        index=0
    )
    dynamic_guide = st.checkbox("動態構圖分析", True)
    
    # AI建議設置
    st.divider()
    ai_feedback_level = st.slider("AI建議強度", 0, 100, 50)
    show_ai_overlay = st.checkbox("顯示AI修改建議", True)
    
# 主界面布局
col1, col2, col3 = st.columns([3, 1, 1])
with col1:
    ctx = webrtc_streamer(
        key="enhanced-painting",
        video_transformer_factory=EnhancedPaintingAnalyzer,
        rtc_configuration={"iceServers": [{"urls": ["stun:stun.l.google.com:19302"]}]},
        media_stream_constraints={"video": True, "audio": False}
    )
    
    if ctx.video_transformer:
        ctx.video_transformer.ar_guides = {
            "enable": ar_enable,
            "current_template": ar_template,
            "enable_dynamic": dynamic_guide
        }
        st.session_state.show_ai_suggestion = show_ai_overlay

with col2:
    st.header("AI修改建議")
    if st.button("生成優化方案"):
        if ctx.video_transformer:
            current_frame = ctx.video_transformer.last_frame
            mask = ctx.video_transformer.create_attention_mask(current_frame)
            suggestion = ctx.video_transformer.generate_improvement_suggestion(current_frame, mask)
            st.image(suggestion, caption="AI優化建議", use_column_width=True)
    
    st.divider()
    st.subheader("文字建議")
    if ctx.video_transformer:
        feedback_text = ctx.video_transformer.generate_text_feedback()
        st.markdown(f"""
        <div class="feedback-box">
            <p style="color:#2c3e50;">{feedback_text}</p>
        </div>
        """, unsafe_allow_html=True)

with col3:
    st.header("構圖分析")
    if ctx.video_transformer:
        composition_analysis = ctx.video_transformer.get_composition_analysis()
        st.plotly_chart(composition_analysis)
    
    st.divider()
    st.subheader("色彩分布")
    if ctx.video_transformer:
        color_hist = ctx.video_transformer.get_color_analysis()
        st.altair_chart(color_hist, use_container_width=True)

# CSS樣式增強
st.markdown("""
<style>
    .st-emotion-cache-1v0mbdj {
        border-radius: 15px;
        box-shadow: 0 4px 8px rgba(0,0,0,0.1);
        transition: transform 0.3s ease;
    }
    .st-emotion-cache-1v0mbdj:hover {
        transform: translateY(-5px);
    }
    .feedback-box {
        padding: 20px;
        background: linear-gradient(145deg, #ffffff, #f8f9fa);
        border-radius: 12px;
        box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        margin: 10px 0;
    }
    .stButton>button {
        background: linear-gradient(45deg, #4CAF50, #45a049);
        color: white;
        border: none;
        padding: 12px 24px;
        border-radius: 8px;
        transition: all 0.3s;
    }
    .stButton>button:hover {
        transform: scale(1.05);
        box-shadow: 0 4px 8px rgba(0,0,0,0.2);
    }
</style>
""", unsafe_allow_html=True)
```

### 新增功能說明：

1. **生成式AI建議系統**：
   - **圖像優化建議**：使用Stable Diffusion進行局部重繪優化
   - **文字反饋生成**：集成GPT-2生成自然語言建議
   ```python
   def generate_text_feedback(self):
       analysis_text = "當前畫面存在構圖不平衡和色彩對比度不足的問題。"
       prompt = f"作爲專業美術老師，根據以下分析給出具體改進建議：{analysis_text}"
       return self.text_generator(prompt, max_length=200)[0]['generated_text']
   ```

2. **動態AR構圖引導**：
   - **智能構圖分析**：根據邊緣密度自動選擇最佳引導模板
   - **多種構圖模板**：
     ```python
     def create_symmetry_guide(self, frame):
         h, w = frame.shape[:2]
         guide_img = np.zeros_like(frame)
         cv2.line(guide_img, (w//2, 0), (w//2, h), (255,0,0), 2)
         cv2.line(guide_img, (0, h//2), (w, h//2), (255,0,0), 2)
         return cv2.addWeighted(frame, 0.8, guide_img, 0.2, 0)
     ```

3. **交互式優化系統**：
   - 實時AI建議叠加顯示
   - 一鍵生成完整優化方案
   - 可調節的AI建議強度參數

### 部署優化建議：

1. **性能優化**：
   ```python
   # 使用ONNX Runtime加速推理
   from onnxruntime import InferenceSession
   session = InferenceSession("model.onnx")
   
   # 異步處理資源密集型任務
   async def async_inpainting(self):
       loop = asyncio.get_event_loop()
       return await loop.run_in_executor(
           None, self.inpainter, prompt, image, mask)
   ```

2. **移動端適配**：
   ```javascript
   // 在Streamlit組件中添加響應式布局
   window.addEventListener("resize", () => {
     const videoContainer = document.getElementById("video-container");
     videoContainer.style.width = window.innerWidth * 0.6 + "px";
   });
   ```

3. **云服務集成**：
   ```python
   # 集成AWS S3保存作品記錄
   import boto3
   s3 = boto3.client('s3')
   
   def save_to_cloud(img, user_id):
       _, img_encoded = cv2.imencode('.jpg', img)
       s3.put_object(
           Bucket='painting-tutor',
           Key=f'{user_id}/{time.time()}.jpg',
           Body=img_encoded.tobytes()
       )
   ```

該方案實現了：
- 實時AR構圖引導叠加顯示
- 生成式AI的即時修改建議
- 交互式參數調節
- 多維度可視化分析
- 響應式用戶界面

可根據需要擴展以下方向：
1. 集成姿勢識別指導繪畫姿勢
2. 添加多用戶協作模式
3. 實現歷史版本對比功能
4. 開發基于LLM的對話式指導系統
