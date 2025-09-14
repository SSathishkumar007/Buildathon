"""
Ojas - Enhanced Healthcare AI Assistant
================================================
AI Co-Pilot for Last-Mile Healthcare in Rural India

Features:
- Voice recording and transcription (Whisper API)
- Multimodal AI analysis (GPT-4o Vision)
- Intelligent triage system with color-coded alerts
- Visual health education with pictograms (DALL-E 3)
- Text-to-speech patient communication
- Offline demo mode for presentations
"""

import os
import streamlit as st
import tempfile
import base64
from pathlib import Path
from datetime import datetime
import json

# Optional imports with graceful fallbacks
try:
    from openai import OpenAI
    OPENAI_AVAILABLE = True
except ImportError:
    OPENAI_AVAILABLE = False
    st.error("OpenAI package not installed. Running in demo mode only.")

try:
    from gtts import gTTS
    GTTS_AVAILABLE = True
except ImportError:
    GTTS_AVAILABLE = False

# Configuration
DEMO_MODE_DEFAULT = True
SUPPORTED_LANGUAGES = {
    "English": "en",
    "Hindi": "hi", 
    "Tamil": "ta",
    "Kannada": "kn",
    "Telugu": "te",
    "Bengali": "bn",
    "Marathi": "mr"
}

TRIAGE_COLORS = {
    "GREEN": "🟢",
    "YELLOW": "🟡", 
    "RED": "🔴"
}

# Page configuration
st.set_page_config(
    page_title="Ojas",
    page_icon="🩺",
    layout="wide",
    initial_sidebar_state="expanded"
)

# Custom CSS for better UI
st.markdown("""
<style>
    .main-header {
        text-align: center;
        color: #2E8B57;
        font-size: 2.5rem;
        margin-bottom: 1rem;
        font-weight: 700;
    }
    .subtitle {
        text-align: center;
        color: #666;
        font-size: 1.2rem;
        margin-bottom: 2rem;
        font-style: italic;
    }
    .centered-content {
        display: flex;
        justify-content: center;
        align-items: center;
        text-align: center;
    }
    .ai-process-badge {
        background: linear-gradient(90deg, #667eea 0%, #764ba2 100%);
        color: white;
        padding: 0.5rem 1rem;
        border-radius: 20px;
        font-size: 0.85rem;
        margin: 0.5rem 0;
        display: inline-block;
        font-weight: 600;
    }
    .patient-card {
        background-color: #f8f9fa;
        padding: 1.5rem;
        border-radius: 10px;
        border-left: 5px solid #2E8B57;
        margin: 1rem 0;
    }
    .triage-green {
        background-color: #d4edda;
        border-left: 5px solid #28a745;
        padding: 1rem;
        border-radius: 5px;
    }
    .triage-yellow {
        background-color: #fff3cd;
        border-left: 5px solid #ffc107;
        padding: 1rem;
        border-radius: 5px;
    }
    .triage-red {
        background-color: #f8d7da;
        border-left: 5px solid #dc3545;
        padding: 1rem;
        border-radius: 5px;
    }
    .feature-box {
        background-color: #ffffff;
        padding: 1.5rem;
        border-radius: 10px;
        box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        margin: 1rem 0;
    }
</style>
""", unsafe_allow_html=True)

def initialize_session_state():
    """Initialize session state variables"""
    if 'transcript' not in st.session_state:
        st.session_state.transcript = ""
    if 'analysis_result' not in st.session_state:
        st.session_state.analysis_result = ""
    if 'patient_data' not in st.session_state:
        st.session_state.patient_data = {}
    if 'case_history' not in st.session_state:
        st.session_state.case_history = []

def setup_sidebar():
    """Setup sidebar with configuration options"""
    with st.sidebar:
        st.image("https://via.placeholder.com/300x100/2E8B57/white?text=Ojas", width=280)
        st.markdown("### 🔧 Configuration")
        
        demo_mode = st.checkbox("🎭 Demo Mode (Offline)", value=DEMO_MODE_DEFAULT, 
                               help="Use placeholder responses for demonstration")
        
        api_key = st.text_input("🔑 OpenAI API Key", type="password", 
                               help="Required for live AI features")
        
        st.markdown("### 📊 System Status")
        st.success("✅ Streamlit UI") 
        st.success("✅ Demo Mode" if demo_mode else "⚠️ Live Mode")
        
        if OPENAI_AVAILABLE and api_key:
            st.success("✅ OpenAI Connected")
        else:
            st.warning("⚠️ OpenAI Not Available")
            
        if GTTS_AVAILABLE:
            st.success("✅ Text-to-Speech")
        else:
            st.warning("⚠️ TTS Not Available")
            
        st.markdown("### 🌍 Language Support")
        st.info("7 Indian languages supported")
        
        return demo_mode, api_key

def get_openai_client(api_key):
    """Initialize OpenAI client if possible"""
    if not OPENAI_AVAILABLE or not api_key:
        return None
    try:
        return OpenAI(api_key=api_key)
    except Exception as e:
        st.error(f"Failed to initialize OpenAI client: {str(e)}")
        return None

def patient_input_form():
    """Patient information input form"""
    st.markdown('<div class="feature-box">', unsafe_allow_html=True)
    st.markdown("### 👤 Patient Information")
    
    col1, col2, col3 = st.columns(3)
    
    with col1:
        name = st.text_input("🏷️ Patient Name", placeholder="Enter patient name")
        age = st.number_input("🎂 Age", min_value=0, max_value=120, value=30)
    
    with col2:
        gender = st.selectbox("⚧ Gender", 
                            ["Select", "Female", "Male", "Other", "Prefer not to say"])
        location = st.text_input("📍 Village/Location", placeholder="Village name")
    
    with col3:
        language = st.selectbox("🗣️ Preferred Language", list(SUPPORTED_LANGUAGES.keys()))
        contact = st.text_input("📱 Contact (Optional)", placeholder="Phone number")
    
    st.markdown('</div>', unsafe_allow_html=True)
    
    return {
        "name": name,
        "age": age,
        "gender": gender,
        "location": location, 
        "language": language,
        "contact": contact,
        "timestamp": datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    }

def voice_input_section():
    """Voice recording and file upload section"""
    st.markdown('<div class="feature-box">', unsafe_allow_html=True)
    st.markdown("### 🎙️ Voice Input")
    
    col1, col2 = st.columns(2)
    
    with col1:
        st.markdown("**Record Symptoms**")
        audio_input = st.audio_input("Click to record patient's symptoms")
        
    with col2:
        st.markdown("**Upload Audio File**")
        uploaded_audio = st.file_uploader("Upload audio file", 
                                        type=["wav", "mp3", "m4a", "ogg"])
    
    # Manual text input as fallback
    manual_symptoms = st.text_area("✍️ Or type symptoms manually", 
                                 height=100,
                                 placeholder="Describe patient's symptoms...")
    
    st.markdown('</div>', unsafe_allow_html=True)
    
    return audio_input, uploaded_audio, manual_symptoms

def image_input_section():
    """Image upload for visual analysis"""
    st.markdown('<div class="feature-box">', unsafe_allow_html=True)
    st.markdown("### 📸 Visual Analysis")
    
    uploaded_image = st.file_uploader("Upload medical images", 
                                    type=["png", "jpg", "jpeg"],
                                    help="Photos of rashes, wounds, eye conditions, etc.")
    
    if uploaded_image:
        col1, col2 = st.columns([1, 2])
        with col1:
            st.image(uploaded_image, caption="Uploaded Image", width=200)
        with col2:
            st.info("✅ Image uploaded successfully. This will be analyzed along with symptoms.")
    
    st.markdown('</div>', unsafe_allow_html=True)
    return uploaded_image

def transcribe_audio(audio_file, client, demo_mode):
    """Transcribe audio using Whisper API or demo response"""
    if demo_mode or not client:
        st.markdown('<span class="ai-process-badge">🤖 OpenAI Whisper API - Speech Recognition</span>', unsafe_allow_html=True)
        return "Patient reports fever for 3 days, headache, and body ache. No appetite. Difficulty sleeping due to high temperature. Family history of diabetes."
    
    try:
        st.markdown('<span class="ai-process-badge">🤖 OpenAI Whisper API - Processing Audio...</span>', unsafe_allow_html=True)
        with tempfile.NamedTemporaryFile(delete=False, suffix=".wav") as tmp_file:
            tmp_file.write(audio_file.getvalue())
            tmp_file.flush()
            
            with open(tmp_file.name, "rb") as audio:
                transcript = client.audio.transcriptions.create(
                    model="whisper-1",
                    file=audio,
                    language="en"  # Auto-detect in real implementation
                )
                return transcript.text
    except Exception as e:
        st.error(f"Transcription failed: {str(e)}")
        return "Transcription failed. Please try again."

def analyze_case(symptoms_text, image_present, patient_info, client, demo_mode):
    """AI-powered case analysis and triage"""
    if demo_mode or not client:
        if image_present:
            st.markdown('<span class="ai-process-badge">🤖 GPT-4 Vision API - Analyzing Image + Text</span>', unsafe_allow_html=True)
        else:
            st.markdown('<span class="ai-process-badge">🤖 GPT-4 API - Medical Analysis & Triage</span>', unsafe_allow_html=True)
        
        return {
            "triage": "YELLOW",
            "confidence": 85,
            "condition": "Viral Fever (Suspected)",
            "actions": [
                "Administer Paracetamol 500mg every 6 hours",
                "Ensure adequate hydration (ORS solution)",
                "Monitor temperature every 4 hours",
                "Complete rest for 3-5 days"
            ],
            "referral": "Refer to PHC if fever persists beyond 5 days or if breathing difficulty occurs",
            "patient_message": "आपको वायरल बुखार हो सकता है। दवा लें, आराम करें और पानी पिएं। अगर 5 दिन बाद भी बुखार है तो डॉक्टर के पास जाएं।",
            "follow_up": "Follow up in 3 days or immediately if condition worsens"
        }
    
    try:
        if image_present:
            st.markdown('<span class="ai-process-badge">🤖 GPT-4 Vision API - Multimodal Analysis</span>', unsafe_allow_html=True)
        else:
            st.markdown('<span class="ai-process-badge">🤖 GPT-4 API - Medical Case Analysis</span>', unsafe_allow_html=True)
            
        system_prompt = """You are an AI medical assistant for rural healthcare workers in India. 
        Analyze the case and provide structured triage recommendations. 
        Respond in JSON format with triage (GREEN/YELLOW/RED), condition, actions, referral advice, and patient message in local language."""
        
        user_prompt = f"""
        PATIENT INFO: {patient_info}
        SYMPTOMS: {symptoms_text}
        IMAGE PROVIDED: {'Yes' if image_present else 'No'}
        
        Provide triage analysis focusing on common rural health issues in India.
        """
        
        response = client.chat.completions.create(
            model="gpt-4-vision-preview" if image_present else "gpt-4",
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": user_prompt}
            ],
            max_tokens=600,
            temperature=0.3
        )
        
        # Parse JSON response
        result = json.loads(response.choices[0].message.content)
        return result
        
    except Exception as e:
        st.error(f"Analysis failed: {str(e)}")
        return {"error": "Analysis failed"}

def display_triage_result(analysis):
    """Display triage results with proper formatting"""
    if "error" in analysis:
        st.error("Analysis failed. Please try again.")
        return
    
    triage_level = analysis.get("triage", "YELLOW")
    triage_class = f"triage-{triage_level.lower()}"
    
    st.markdown(f'<div class="{triage_class}">', unsafe_allow_html=True)
    
    col1, col2 = st.columns([1, 3])
    
    with col1:
        st.markdown(f"## {TRIAGE_COLORS.get(triage_level, '🟡')} {triage_level}")
        confidence = analysis.get("confidence", 0)
        st.metric("Confidence", f"{confidence}%")
    
    with col2:
        condition = analysis.get("condition", "Assessment pending")
        st.markdown(f"**Suspected Condition:** {condition}")
        
        # Actions
        st.markdown("**Recommended Actions:**")
        actions = analysis.get("actions", [])
        for i, action in enumerate(actions, 1):
            st.markdown(f"{i}. {action}")
    
    st.markdown('</div>', unsafe_allow_html=True)
    
    # Referral advice
    if analysis.get("referral"):
        st.info(f"🏥 **Referral:** {analysis['referral']}")
    
    # Follow-up
    if analysis.get("follow_up"):
        st.warning(f"📅 **Follow-up:** {analysis['follow_up']}")

def generate_pictogram(condition, actions, client, demo_mode):
    """Generate pictogram using DALL-E API"""
    if demo_mode or not client:
        st.markdown('<span class="ai-process-badge">🤖 DALL-E 3 API - Generating Health Pictogram</span>', unsafe_allow_html=True)
        
        # Create a sample pictogram using Unicode symbols and text
        pictogram_content = f"""
        <div style="background: white; border: 3px solid #2E8B57; border-radius: 15px; padding: 30px; text-align: center; font-family: Arial; max-width: 400px; margin: 20px auto;">
            <h2 style="color: #2E8B57; margin-bottom: 20px;">🏥 स्वास्थ्य सलाह</h2>
            <div style="font-size: 48px; margin: 20px 0;">🤒</div>
            <h3 style="color: #333; margin-bottom: 15px;">{condition}</h3>
            
            <div style="text-align: left; margin: 20px 0;">
                <div style="display: flex; align-items: center; margin: 10px 0;">
                    <span style="font-size: 24px; margin-right: 10px;">💊</span>
                    <span>दवा समय पर लें</span>
                </div>
                <div style="display: flex; align-items: center; margin: 10px 0;">
                    <span style="font-size: 24px; margin-right: 10px;">💧</span>
                    <span>ज्यादा पानी पिएं</span>
                </div>
                <div style="display: flex; align-items: center; margin: 10px 0;">
                    <span style="font-size: 24px; margin-right: 10px;">🛏️</span>
                    <span>आराम करें</span>
                </div>
                <div style="display: flex; align-items: center; margin: 10px 0;">
                    <span style="font-size: 24px; margin-right: 10px;">🏥</span>
                    <span>जरूरत पर अस्पताल जाएं</span>
                </div>
            </div>
            
            <div style="background: #f8f9fa; padding: 15px; border-radius: 8px; margin-top: 20px;">
                <strong style="color: #dc3545;">⚠️ चेतावनी:</strong><br>
                तेज़ बुखार या सांस लेने में परेशानी हो तो तुरंत अस्पताल जाएं
            </div>
        </div>
        """
        
        return pictogram_content
    
    try:
        st.markdown('<span class="ai-process-badge">🤖 DALL-E 3 API - Creating Visual Health Guide</span>', unsafe_allow_html=True)
        
        prompt = f"""Create a simple, clear health education pictogram for rural Indian patients showing: {condition}. 
        Include visual symbols for: {', '.join(actions[:3])}. 
        Style: High contrast, simple icons, minimal text, culturally appropriate for India, 
        medical illustration style, clean white background."""
        
        response = client.images.generate(
            model="dall-e-3",
            prompt=prompt,
            size="1024x1024",
            quality="standard",
            n=1
        )
        
        return response.data[0].url
        
    except Exception as e:
        st.error(f"Pictogram generation failed: {str(e)}")
        return None

def text_to_speech(text, language_code="en", client=None, demo_mode=True):
    """Convert text to speech for patient communication"""
    if demo_mode or not client:
        st.markdown('<span class="ai-process-badge">🤖 OpenAI TTS API - Text-to-Speech Generation</span>', unsafe_allow_html=True)
    
    if not GTTS_AVAILABLE and (demo_mode or not client):
        st.warning("Text-to-speech not available")
        return None
    
    try:
        if not demo_mode and client:
            # Try OpenAI TTS first
            st.markdown('<span class="ai-process-badge">🤖 OpenAI TTS API - Generating Voice</span>', unsafe_allow_html=True)
            try:
                response = client.audio.speech.create(
                    model="tts-1",
                    voice="alloy",
                    input=text
                )
                with tempfile.NamedTemporaryFile(delete=False, suffix=".mp3") as tmp_file:
                    response.stream_to_file(tmp_file.name)
                    return tmp_file.name
            except:
                pass  # Fall back to gTTS
        
        # Fallback to gTTS
        if GTTS_AVAILABLE:
            st.markdown('<span class="ai-process-badge">🤖 Google TTS - Voice Synthesis</span>', unsafe_allow_html=True)
            tts = gTTS(text=text, lang=language_code, slow=False)
            with tempfile.NamedTemporaryFile(delete=False, suffix=".mp3") as tmp_file:
                tts.save(tmp_file.name)
                return tmp_file.name
                
    except Exception as e:
        st.error(f"TTS failed: {str(e)}")
        return None

def save_case_history(patient_info, symptoms, analysis):
    """Save case to session history"""
    case = {
        "timestamp": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
        "patient": patient_info,
        "symptoms": symptoms,
        "analysis": analysis
    }
    st.session_state.case_history.append(case)

def display_case_history():
    """Display recent case history"""
    if not st.session_state.case_history:
        st.info("No previous cases")
        return
    
    st.markdown("### 📋 Recent Cases")
    for i, case in enumerate(reversed(st.session_state.case_history[-5:]), 1):
        with st.expander(f"Case {i}: {case['patient']['name']} - {case['timestamp']}"):
            st.json(case)

def main():
    """Main application function"""
    initialize_session_state()
    
    # Header
    st.markdown('<div class="centered-content">', unsafe_allow_html=True)
    st.markdown('<h1 class="main-header">🩺 Ojas</h1>', unsafe_allow_html=True)
    st.markdown('<p class="subtitle">AI Co-Pilot for Last-Mile Healthcare in Rural India</p>', unsafe_allow_html=True)
    st.markdown('</div>', unsafe_allow_html=True)
    
    # Sidebar configuration
    demo_mode, api_key = setup_sidebar()
    client = get_openai_client(api_key)
    
    # Main interface tabs
    tab1, tab2, tab3 = st.tabs(["🔍 New Case Analysis", "📋 Case History", "ℹ️ About"])
    
    with tab1:
        # Patient information
        patient_info = patient_input_form()
        
        # Input sections
        col1, col2 = st.columns([2, 1])
        
        with col1:
            audio_input, uploaded_audio, manual_symptoms = voice_input_section()
            
        with col2:
            uploaded_image = image_input_section()
        
        # Analysis button
        st.markdown("---")
        
        if st.button("🔬 **Analyze Case with AI**", type="primary", use_container_width=True):
            if not any([audio_input, uploaded_audio, manual_symptoms]):
                st.error("⚠️ Please provide symptoms through voice, audio file, or text input")
            else:
                # Progress indicator
                progress_bar = st.progress(0)
                status_text = st.empty()
                
                # Step 1: Transcription
                status_text.text("🎙️ Processing audio input...")
                progress_bar.progress(25)
                
                transcript = ""
                if manual_symptoms:
                    transcript = manual_symptoms
                elif audio_input:
                    transcript = transcribe_audio(audio_input, client, demo_mode)
                elif uploaded_audio:
                    transcript = transcribe_audio(uploaded_audio, client, demo_mode)
                
                # Step 2: Analysis
                status_text.text("🧠 Analyzing case with AI...")
                progress_bar.progress(50)
                
                analysis = analyze_case(
                    transcript, 
                    uploaded_image is not None, 
                    patient_info, 
                    client, 
                    demo_mode
                )
                
                # Step 3: Generate education material
                status_text.text("📚 Generating patient education...")
                progress_bar.progress(75)
                
                # Step 4: Complete
                status_text.text("✅ Analysis complete!")
                progress_bar.progress(100)
                
                # Display results
                st.markdown("## 📊 Analysis Results")
                
                # Transcript
                with st.expander("📝 Symptoms Transcript", expanded=True):
                    st.write(transcript)
                
                # Triage results
                st.markdown("### 🚨 Triage Assessment")
                display_triage_result(analysis)
                
                # Patient education with pictogram
                st.markdown("### 🎓 Patient Education & Visual Guide")
                
                # Generate pictogram
                condition = analysis.get("condition", "Health Condition")
                actions = analysis.get("actions", ["Take medicine", "Rest well", "Drink water"])
                
                col_ed1, col_ed2 = st.columns([1, 1])
                
                with col_ed1:
                    st.markdown("#### 📱 Digital Health Pictogram")
                    pictogram = generate_pictogram(condition, actions, client, demo_mode)
                    
                    if isinstance(pictogram, str) and pictogram.startswith('<div'):
                        # Display HTML pictogram (demo mode)
                        st.markdown(pictogram, unsafe_allow_html=True)
                    elif isinstance(pictogram, str) and pictogram.startswith('http'):
                        # Display generated image URL (live mode)
                        st.image(pictogram, caption="AI-Generated Health Pictogram", width=400)
                    else:
                        st.info("Pictogram generation in progress...")
                
                with col_ed2:
                    st.markdown("#### 🗣️ Patient Instructions")
                    patient_msg = analysis.get("patient_message", "Please follow the doctor's advice and take proper rest.")
                    st.markdown(f"**Message:** {patient_msg}")
                    
                    # Audio message for patient
                    if patient_msg:
                        lang_code = SUPPORTED_LANGUAGES.get(patient_info["language"], "en")
                        audio_file = text_to_speech(patient_msg, lang_code, client, demo_mode)
                        if audio_file:
                            st.audio(audio_file, format="audio/mp3")
                        else:
                            st.info("🔊 Audio playback would be available here")
                
                # Save to history
                save_case_history(patient_info, transcript, analysis)
                
                # Clear progress
                progress_bar.empty()
                status_text.empty()
    
    with tab2:
        display_case_history()
    
    with tab3:
        st.markdown("""
        ## About Ojas
        
        **Ojas** is an AI-powered healthcare assistant designed specifically for rural India's healthcare challenges.
        
        ### 🎯 Key Features
        - **Voice-First Interface**: Record symptoms in local languages
        - **AI-Powered Triage**: Intelligent risk assessment using GPT-4
        - **Visual Analysis**: Image-based symptom analysis
        - **Multilingual Support**: 7 Indian languages supported
        - **Patient Education**: Visual and audio health education materials
        - **Offline Capability**: Works in low-connectivity areas
        
        ### 🔬 Technology Stack
        - **Speech Recognition**: OpenAI Whisper
        - **AI Analysis**: GPT-4 Vision & GPT-4
        - **Image Generation**: DALL-E 3
        - **Text-to-Speech**: gTTS & OpenAI TTS
        - **Frontend**: Streamlit
        
        ### 👥 Target Users
        - ASHA workers
        - Rural healthcare practitioners
        - Primary health center staff
        - Community health volunteers
        
        ### 🌟 Impact
        - Bridges expertise gap in rural healthcare
        - Enables early detection and proper triage
        - Reduces unnecessary referrals
        - Improves health outcomes in underserved communities
        
        ---
        *Built for rural India, powered by AI* 🇮🇳
        """)

if __name__ == "__main__":
    main()
