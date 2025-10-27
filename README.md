# test2
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>초등학생 수행평가 관리</title>
    <!-- Tailwind CSS CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap');
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f3f4f6;
        }
        .container {
            max-width: 1200px;
            margin: 0 auto;
        }
        .drop-zone {
            border: 2px dashed #9ca3af;
            background-color: #e5e7eb;
            transition: all 0.3s ease;
        }
        .drop-zone-over {
            border-color: #1f2937;
            background-color: #d1d5db;
        }
        /* 답안지 컨테이너를 더 넓게 조정 */
        #answer-preview-container {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
            gap: 20px;
        }
        .criteria-preview-container {
             display: grid;
            grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
            gap: 16px;
        }
        .image-preview-item {
            position: relative;
            background-color: white; /* 답안지 아이템 배경색 */
            padding: 12px;
            border-radius: 12px;
            box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
        }
        .delete-btn {
            position: absolute;
            top: 4px;
            right: 4px;
            background-color: rgba(255, 255, 255, 0.9);
            border-radius: 50%;
            padding: 4px;
            cursor: pointer;
            transition: all 0.2s ease;
        }
        .delete-btn:hover {
            background-color: rgba(255, 255, 255, 1);
        }
        .loading-spinner {
            border-top-color: transparent;
            border-radius: 50%;
            animation: spin 1s linear infinite;
        }
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
    </style>
</head>
<body class="bg-gray-100 p-8">

    <div class="container bg-white p-8 rounded-2xl shadow-xl space-y-8">
        <h1 class="text-4xl font-bold text-center text-gray-800 mb-6">초등학생 수행평가 관리 웹앱</h1>

        <!-- Global Inputs -->
        <div class="bg-indigo-50 p-4 rounded-xl border border-indigo-200 flex items-center space-x-4">
            <label for="subject" class="text-gray-700 font-medium whitespace-nowrap">평가 과목:</label>
            <input type="text" id="subject" class="flex-1 p-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-indigo-500" placeholder="예) 국어">
        </div>

        <!-- File Upload Section for Criteria -->
        <div class="bg-yellow-50 p-6 rounded-xl border border-yellow-200">
            <h2 class="text-2xl font-semibold text-gray-700 mb-4">평가 기준 및 성취기준 사진 업로드 (필수)</h2>
            <div id="criteria-drop-zone" class="drop-zone flex flex-col items-center justify-center p-8 rounded-xl text-gray-500">
                <p class="text-lg font-medium">평가 기준 사진을 여기에 드래그하거나 클릭하여 업로드하세요</p>
                <p class="text-sm mt-1">(하나의 기준 파일만 허용)</p>
                <input type="file" id="criteria-upload" class="hidden" accept="image/jpeg, image/png">
            </div>
            <div id="criteria-preview-container" class="criteria-preview-container mt-6"></div>
        </div>

        <!-- File Upload Section for Student Answer -->
        <div class="bg-blue-50 p-6 rounded-xl border border-blue-200">
            <h2 class="text-2xl font-semibold text-gray-700 mb-4">학생 답안지 사진 일괄 업로드</h2>
            <div id="answer-drop-zone" class="drop-zone flex flex-col items-center justify-center p-8 rounded-xl text-gray-500">
                <p class="text-lg font-medium">학생 답안지 사진들을 여기에 드래그하거나 클릭하여 업로드하세요</p>
                <p class="text-sm mt-1">(여러 파일을 한 번에 업로드 가능)</p>
                <input type="file" id="answer-upload" class="hidden" multiple accept="image/jpeg, image/png">
            </div>
            <!-- 개별 학생별 답안지 및 평가 컨트롤이 여기에 표시됩니다 -->
            <div id="answer-preview-container" class="mt-6"></div>
        </div>
        
        <!-- Notification Message Box -->
        <div id="notification" class="fixed bottom-8 right-8 bg-blue-600 text-white px-6 py-3 rounded-lg shadow-lg hidden z-50">
            메시지
        </div>
    </div>

    <script>
        // DOM 요소 가져오기
        const criteriaDropZone = document.getElementById('criteria-drop-zone');
        const criteriaUpload = document.getElementById('criteria-upload');
        const criteriaPreviewContainer = document.getElementById('criteria-preview-container');
        const answerDropZone = document.getElementById('answer-drop-zone');
        const answerUpload = document.getElementById('answer-upload');
        const answerPreviewContainer = document.getElementById('answer-preview-container');
        const subjectInput = document.getElementById('subject');
        const notification = document.getElementById('notification');
        
        // 고정된 API 키
        const apiKey = "AIzaSyCWpt1uxNSdV2B_QKV5B1G7Vd6osT3xCIc";
        const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-05-20:generateContent?key=${apiKey}`;

        // 파일 드래그 앤 드롭 이벤트 등록 (criteria, answer 공통)
        function registerDropZone(dropZone, uploadInput, container, isSingle = false) {
            dropZone.addEventListener('dragover', (e) => {
                e.preventDefault();
                dropZone.classList.add('drop-zone-over');
            });

            dropZone.addEventListener('dragleave', (e) => {
                e.preventDefault();
                dropZone.classList.remove('drop-zone-over');
            });

            dropZone.addEventListener('drop', (e) => {
                e.preventDefault();
                dropZone.classList.remove('drop-zone-over');
                handleFiles(e.dataTransfer.files, container, isSingle);
            });

            dropZone.addEventListener('click', () => {
                uploadInput.click();
            });

            uploadInput.addEventListener('change', (e) => {
                handleFiles(e.target.files, container, isSingle);
            });
        }
        
        registerDropZone(criteriaDropZone, criteriaUpload, criteriaPreviewContainer, true); // 기준은 하나만
        registerDropZone(answerDropZone, answerUpload, answerPreviewContainer);             // 답안은 여러 개

        function handleFiles(files, container, isSingle) {
            Array.from(files).forEach(file => {
                if (file.type.startsWith('image/')) {
                    const reader = new FileReader();
                    reader.onload = (e) => {
                        if (isSingle) {
                            // 기준 파일은 하나만 유지
                            container.innerHTML = ''; 
                            createImagePreview(e.target.result, container, isSingle);
                        } else {
                            createImagePreview(e.target.result, container, isSingle);
                        }
                    };
                    reader.readAsDataURL(file);
                } else {
                    showNotification('이미지 파일만 업로드할 수 있습니다.');
                }
            });
        }

        function createImagePreview(imageDataUrl, container, isSingle) {
            const previewItem = document.createElement('div');
            previewItem.className = 'image-preview-item relative group';
            
            if (isSingle) {
                 // 기준 이미지 (간단한 미리보기)
                previewItem.className = 'image-preview-item relative group p-0 bg-transparent';
                previewItem.innerHTML = `
                    <img src="${imageDataUrl}" alt="criteria image" class="rounded-lg shadow-md w-full h-auto object-cover border border-gray-300">
                    <button class="delete-btn opacity-100 transition-opacity duration-200" title="삭제">
                        <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 text-gray-600" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
                        </svg>
                    </button>
                `;
            } else {
                // 학생 답안 이미지 (이름 입력 및 생성 버튼 포함)
                previewItem.className = 'image-preview-item relative group bg-white p-4 rounded-xl border border-blue-300 space-y-3';
                previewItem.innerHTML = `
                    <img src="${imageDataUrl}" alt="student answer" class="rounded-lg shadow-sm w-full h-auto object-cover mb-3">
                    <input type="text" placeholder="학생 이름 입력 (예: 김민지)" class="student-name-input w-full p-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500">
                    
                    <button class="generate-student-opinion-btn w-full bg-green-600 text-white text-sm font-semibold py-2 rounded-xl shadow-md hover:bg-green-700 transition duration-300 disabled:opacity-50 disabled:cursor-not-allowed flex items-center justify-center">
                        <span class="btn-text">종합 의견 생성</span>
                        <div class="loading-spinner hidden w-4 h-4 border-2 border-white"></div>
                    </button>
                    
                    <div class="generated-opinion-result mt-3 p-3 bg-gray-50 border border-gray-200 rounded-lg min-h-[50px] text-xs whitespace-pre-wrap">
                        결과 대기 중...
                    </div>
                    <button class="copy-result-btn absolute top-14 right-6 bg-gray-200 p-1 rounded-full shadow-md hover:bg-gray-300 transition duration-200 hidden" title="결과 복사">
                        <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 text-gray-600" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 16H6a2 2 0 01-2-2V6a2 2 0 012-2h8a2 2 0 012 2v2m-6 12h8a2 2 0 002-2v-8a2 2 0 00-2-2h-8a2 2 0 00-2 2v8a2 2 0 002 2z" />
                        </svg>
                    </button>
                    <button class="delete-btn opacity-100 transition-opacity duration-200" title="삭제">
                        <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 text-gray-600" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
                        </svg>
                    </button>
                `;
                
                // 이벤트 리스너 추가 (학생 답안지에만 해당)
                const generateButton = previewItem.querySelector('.generate-student-opinion-btn');
                const resultDiv = previewItem.querySelector('.generated-opinion-result');
                const studentNameInput = previewItem.querySelector('.student-name-input');
                const copyButton = previewItem.querySelector('.copy-result-btn');

                generateButton.addEventListener('click', () => {
                    const studentName = studentNameInput.value.trim();
                    generateOpinionForStudent(studentName, imageDataUrl, generateButton, resultDiv, copyButton);
                });

                copyButton.addEventListener('click', () => {
                    copyOpinionToClipboard(resultDiv.textContent);
                });
            }
            
            // 삭제 버튼 이벤트 리스너
            previewItem.querySelector('.delete-btn').addEventListener('click', () => {
                previewItem.remove();
            });
            
            container.appendChild(previewItem);
        }

        // 알림 메시지 표시 함수
        function showNotification(message, duration = 3000) {
            notification.textContent = message;
            notification.classList.remove('hidden');
            setTimeout(() => {
                notification.classList.add('hidden');
            }, duration);
        }
        
        // 클립보드 복사 함수
        function copyOpinionToClipboard(text) {
             if (text && text.trim() !== '결과 대기 중...') {
                try {
                    const textarea = document.createElement('textarea');
                    textarea.value = text;
                    document.body.appendChild(textarea);
                    textarea.select();
                    document.execCommand('copy');
                    document.body.removeChild(textarea);

                    showNotification('종합 의견이 클립보드에 복사되었습니다!');
                } catch (err) {
                    showNotification('복사 중 오류가 발생했습니다.');
                }
            } else {
                showNotification('복사할 내용이 없습니다.');
            }
        }


        // 개별 학생의 종합 의견 생성 함수
        async function generateOpinionForStudent(studentName, answerImageDataUrl, buttonElement, resultDiv, copyButton) {
            const subject = subjectInput.value.trim();
            const criteriaImages = criteriaPreviewContainer.querySelectorAll('img');
            
            // 필수 입력값 및 이미지 확인
            if (!studentName) {
                showNotification('해당 학생의 이름을 입력해주세요.');
                return;
            }
            if (!subject) {
                showNotification('평가 과목을 상단의 입력란에 입력해주세요.');
                return;
            }
            if (criteriaImages.length === 0) {
                showNotification('평가 기준 사진을 먼저 업로드해주세요.');
                return;
            }

            // UI 상태 변경
            buttonElement.disabled = true;
            buttonElement.querySelector('.btn-text').classList.add('hidden');
            buttonElement.querySelector('.loading-spinner').classList.remove('hidden');
            resultDiv.textContent = 'AI가 채점 및 의견을 생성하는 중입니다...';
            copyButton.classList.add('hidden');
            
            // 기준 이미지의 Base64 데이터 추출 (첫 번째 기준 이미지만 사용)
            const criteriaImageBase64 = criteriaImages[0].src.split(',')[1];
            const answerImageBase64 = answerImageDataUrl.split(',')[1];
            
            // 프롬프트 수정: 개별 학생 맞춤형, 등급별 차등 반영 요청
            const prompt = `초등학교 교사로서 학생 '${studentName}'의 '${subject}' 과목 수행평가 종합 의견을 작성해 주세요. 첨부된 첫 번째 이미지는 채점 기준이고, 두 번째 이미지는 학생 답안지입니다.

            **지시사항:**
            1. **채점:** 채점 기준(첫 번째 이미지)과 학생 답안(두 번째 이미지)을 면밀히 비교하여 '매우잘함', '잘함', '보통', '노력요함' 중 해당하는 평가 등급을 먼저 명시해 주세요.
            2. **개별성:** 학생의 답안지에서 발견되는 구체적인 성취도와 특성을 반영하여 개인 맞춤형 의견을 작성해야 합니다.
            3. **차별화:** 평가 등급에 따라 의견의 내용과 강조점을 명확히 차별화해야 합니다.
               - '매우잘함': 성취기준을 완벽하게 달성하고 창의성, 깊이 등 우수성을 부각합니다.
               - '잘함': 성취기준을 충실히 달성했으며, 특정 강점을 구체적인 예시와 함께 서술합니다.
               - '보통': 성취기준을 대체로 달성했지만, 보완할 점을 긍정적인 표현으로 간결하게 언급합니다.
               - '노력요함': 성취기준 도달에 미흡한 부분을 구체적으로 언급하고, 성장을 위한 명확한 조언을 긍정적인 어조로 제시합니다.
            4. **어미 통일:** 종합 의견의 어미는 '~함.' 또는 '~임.'으로 통일해 주세요. 500자 내외로 작성해 주세요.`;

            try {
                // API 호출을 위한 payload 구성 (기준 이미지 1개 + 답안 이미지 1개)
                const payload = {
                    contents: [{
                        role: "user",
                        parts: [
                            { text: prompt },
                            { inlineData: { mimeType: "image/png", data: criteriaImageBase64 } },
                            { inlineData: { mimeType: "image/png", data: answerImageBase64 } }
                        ]
                    }],
                };
                
                const response = await fetch(apiUrl, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(payload)
                });
                
                if (!response.ok) {
                    const errorData = await response.json();
                    throw new Error(`API 오류: ${errorData.error.message}`);
                }

                const result = await response.json();
                
                if (result.candidates && result.candidates.length > 0 &&
                    result.candidates[0].content && result.candidates[0].content.parts &&
                    result.candidates[0].content.parts.length > 0) {
                    const opinionText = result.candidates[0].content.parts[0].text;
                    resultDiv.textContent = opinionText;
                    showNotification(`${studentName} 학생의 종합 의견 생성이 완료되었습니다!`);
                    copyButton.classList.remove('hidden');
                } else {
                    resultDiv.textContent = '종합 의견을 생성하지 못했습니다. 다시 시도해 주세요.';
                    showNotification('종합 의견 생성 실패');
                }

            } catch (error) {
                console.error('API 호출 중 오류 발생:', error);
                resultDiv.textContent = '오류가 발생했습니다. 다시 시도해 주세요.';
                showNotification('API 호출 실패');
            } finally {
                // UI 상태 복원
                buttonElement.disabled = false;
                buttonElement.querySelector('.btn-text').classList.remove('hidden');
                buttonElement.querySelector('.loading-spinner').classList.add('hidden');
            }
        }
    </script>

</body>
</html>
