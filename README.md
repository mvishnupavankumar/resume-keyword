```markdown
# 🤖 Resume Keyword Matcher Workflow

## 📖 Description

This n8n workflow automates the process of extracting keywords from a resume and a job description (JD), comparing them, and generating a match report. It is designed to be triggered via Telegram, accept a resume as an image or PDF, extract text using OCR, extract keywords from both the resume and JD using Gemini 2.0 Flash LLM model, and then send a match report back to the user via Telegram.

**APIs, Integrations, and Credentials:**

- **Telegram API:** Used for receiving resumes and sending match reports. Requires a Telegram Bot Token.
- **Mistral AI OCR API:** Used for extracting text from image-based resumes. Requires an API token.
- **Google Gemini 2.0 Flash API:** Used for extracting keywords from text. Requires a Google API key.

**High-Level Architecture:**

1.  **Telegram Trigger:** Receives a message containing a resume file (image or PDF).
2.  **File Retrieval:** Downloads the resume file from Telegram.
3.  **OCR (if needed):** If the resume is an image, uses Mistral AI OCR to extract the text.
4.  **Keyword Extraction (Resume & JD):** Uses Google Gemini 2.0 Flash to extract keywords from both the resume and a predefined job description.
5.  **Comparison:** Compares the extracted keywords to determine the match percentage and identify missing keywords.
6.  **Report Generation:** Formats the comparison results into a human-readable match report.
7.  **Telegram Notification:** Sends the match report back to the user via Telegram.

## ⚙️ Workflow Logic (Node by Node)

### 🔹 Telegram Trigger 📡

-   **Function**: Listens for new messages in a Telegram chat and triggers the workflow.
-   **Implementation**:
    -   `updates`: `message` (listens for messages).
    -   Credentials: `telegramApi` (requires a Telegram Bot API token stored in n8n credentials).
    -   `webhookId`: Unique ID for the webhook.

### 🔹 Get a file1 📄

-   **Function**: Retrieves a file from Telegram using the file ID received from the Telegram Trigger.
-   **Implementation**:
    -   `resource`: `file`
    -   `fileId`: Expression that extracts the file ID from the Telegram message (`{{ $('Telegram Trigger').item.json.message?.photo ? $('Telegram Trigger').item.json.message.photo.at(-1).file_id : (['application/pdf','image/png','image/jpeg','image/jpg'].includes($('Telegram Trigger').item.json.message?.document?.mime_type) ? $('Telegram Trigger').item.json.message.document.file_id : '') }}`). This handles both images (using `photo.at(-1).file_id`) and documents (using `document.file_id`), supporting image and PDF file types.
    -   Credentials: `telegramApi` (uses the same Telegram Bot API token).

### 🔹 Edit Fields1 ✏️

-   **Function**: Sets the `url` field with the complete URL to access the file.
-   **Implementation**:
    -   `assignments`: Sets the `url` field to `=https://api.telegram.org/file/botYOUR_TELEGRAM_BOT_TOKEN/{{$json.result.file_path}}`.
        -   **Important**: Replace `YOUR_TELEGRAM_BOT_TOKEN` with your actual Telegram Bot Token.

### 🔹 which ocr text reader get text from image 🤖

-   **Function**: Uses the Mistral AI OCR API to extract text from the resume image.
-   **Implementation**:
    -   `method`: `POST`
    -   `url`: `https://api.mistral.ai/v1/ocr`
    -   `sendHeaders`: `true`
    -   `headerParameters`:
        -   `Content-Type`: `application/json`
        -   `Authorization`: `Bearer YOUR_API_TOKEN` (**Important**: Replace `YOUR_API_TOKEN` with your Mistral AI API key).
    -   `sendBody`: `true`
    -   `specifyBody`: `json`
    -   `jsonBody`:
        ```json
        {
          "model": "mistral-ocr-latest",
          "include_image_base64": true,
          "document": {
            "type": "{{ $json.mime_type?.startsWith('image/') ? 'image_url' : 'document_url' }}",
            "document_url": "{{ $json.url }}"
          }
        }
        ```
    -   This dynamically selects `image_url` or `document_url` based on the file's MIME type, and uses the `url` obtained from previous nodes.
    -   `options`: `response` set to `fullResponse: true`, `neverError: true`, and `responseFormat: json`

### 🔹 Edit Fields 📝

-   **Function**: Extracts the OCR'd text from the Mistral AI API response.
-   **Implementation**:
    -   `assignments`: Sets the `TEXT` field to `{{ $json.body.pages[0].markdown }}`.

### 🔹 Code ⚙️

-   **Function**: Cleans the extracted resume and job description texts, removing special characters and converting them to lowercase.
-   **Implementation**:
    -   `jsCode`:
        ```javascript
        const clean = text => text.replace(/[^a-zA-Z0-9\s]/g, "").toLowerCase();

        let resumeRaw = "";
        let jobRaw = "";

        // Dynamically scan all keys to extract resume and job text
        for (const key of Object.keys($json)) {
          const lowerKey = key.toLowerCase();
          const value = $json[key];

          if (typeof value === "string") {
            if (lowerKey.includes("resume") || lowerKey === "text") {
              resumeRaw = value;
            } else if (lowerKey.includes("job")) {
              jobRaw = value;
            }
          }
        }

        // Fallbacks if nothing was found
        resumeRaw = resumeRaw || "";
        jobRaw = jobRaw || "";

        return {
          json: {
            resumeText: clean(resumeRaw),
            jobText: clean(jobRaw)
          }
        };
        ```
    -   This code dynamically identifies `resume` and `job` text from the input data, cleans them, and outputs them as `resumeText` and `jobText`.

### 🔹 Code1 🧹

-   **Function**: Cleans the `resumeText` further by normalizing whitespace.
-   **Implementation**:
    -   `jsCode`:
        ```javascript
        return items.map(item => {
          const raw = item.json.resumeText || '';
          const cleaned = raw
            .replace(/\s+/g, ' ')       // Replace multiple spaces/newlines/tabs with a single space
            .trim();                    // Remove starting/ending whitespace

          return {
            json: {
              ...item.json,
              resumeText: cleaned
            }
          };
        });
        ```
    -  This ensures all whitespace is normalized to single spaces and removes leading/trailing whitespace.

### 🔹 HTTP Request 📡

-   **Function**: Sends the cleaned resume text to the Google Gemini 2.0 Flash API to extract keywords.
-   **Implementation**:
    -   `method`: `POST`
    -   `url`: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`
    -   `sendHeaders`: `true`
    -   `headerParameters`:
        -   `X-goog-api-key`: `YOUR_GOOGLE_API_KEY` (**Important**: Replace `YOUR_GOOGLE_API_KEY` with your Google API key).
        -   `Content-Type`: `application/json`
    -   `sendBody`: `true`
    -   `specifyBody`: `json`
    -   `jsonBody`:
        ```json
        {
          "contents": [
            {
              "parts": [
                {
                  "text": "Extract the most relevant keywords, skills, and technologies and intelligence seperate by commas  and  as a array from this resume:\\n\\n'{{ $json.resumeText }}'"
                }
              ]
            }
          ]
        }
        ```
    -   `options`: `response` set to `fullResponse: true`, `neverError: true`, and `responseFormat: json`.

### 🔹 Edit Fields4 📝

-   **Function**: Extracts the Gemini's response from the API response and sets it to `DESCRIPTION`.
-   **Implementation**:
    -   `assignments`: Sets the `DESCRIPTION` field to `{{ $json.body.candidates[0].content.parts[0].text }}`.

### 🔹 Edit Fields2 📝

-   **Function**: Sets a sample Job Description to the `description` field.
-   **Implementation**:
    -   `assignments`: Sets the `description` field to a predefined IT Support Specialist job description.
        ```text
        💼 IT Support Specialist / Computer Information Systems Intern – Job Description We are seeking a Computer Information Systems Intern or IT Support Specialist with a solid foundation in system administration, hardware/software troubleshooting, and technical support. The ideal candidate will bring hands-on experience, academic excellence, and a collaborative spirit to support enterprise technology operations and end-user computing.  📌 Responsibilities: Provide technical support for hardware/software issues, including installation, configuration, and troubleshooting  Maintain and administer Windows (19xx/20xx/XP/NT) and UNIX/Linux systems  Assist with the setup and management of local area networks (LAN), including Novell and Windows environments  Staff help desks, analyze reported problems, and resolve user issues efficiently  Contribute to web development projects for internal departmental sites  Support the maintenance of email and other web-based communication systems  Offer technical training and guidance on MS Office and other software tools to university students  📚 Requirements: Pursuing or completed a BS in Computer Information Systems – Web-based Networking  Strong GPA (3.86) and proven academic achievement as a Dean’s Scholar  Knowledge of SQL, PL/SQL, MS Access, and relational databases  Proficiency in Visual Basic, C, C++, Java, HTML, XML, ASP.NET  Experience with tools like MS Office Suite (Word, Excel, PowerPoint, Outlook) and MS Project  Prior internship or support experience in large-scale organizations preferred (e.g., Union Pacific Railroad)  Effective communication and teamwork abilities; demonstrated through student lab assistant and newsletter contributor roles  🎯 Preferred Attributes: Strong problem-solving and analytical skills  Excellent customer service mindset with attention to user needs  Ability to troubleshoot independently and collaboratively  Active involvement in college activities such as technical publications or athletics
        ```
    -   This node serves as input for the Job Description.

### 🔹 HTTP Request1 📡

-   **Function**: Sends the job description text to the Google Gemini 2.0 Flash API to extract keywords.
-   **Implementation**:
    -   `method`: `POST`
    -   `url`: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`
    -   `sendHeaders`: `true`
    -   `headerParameters`:
        -   `X-goog-api-key`: `YOUR_GOOGLE_API_KEY` (**Important**: Replace `YOUR_GOOGLE_API_KEY` with your Google API key).
    -   `sendBody`: `true`
    -   `specifyBody`: `json`
    -   `jsonBody`:
        ```json
        {
          "contents": [
            {
              "parts": [
                {
                  "text": "Extract the most relevant keywords, skills, and technologies and intelligence seperate by commas  and  as a array from this resume:\\n\\n'{{ $json.description }}'"
                }
              ]
            }
          ]
        }
        ```
    -   `options`: `response` set to `fullResponse: true`, `neverError: true`, and `responseFormat: json`.

### 🔹 Edit Fields3 📝

-   **Function**: Extracts the Gemini's JD response from the API response and sets it to `DESCRIPTION`.
-   **Implementation**:
    -   `assignments`: Sets the `DESCRIPTION` field to `{{ $json.body.candidates[0].content.parts[0].text }}`.

### 🔹 Code5 ⚙️

-   **Function**: Parses the Job Description text and extracts keywords from a specified `Keywords` section.
-   **Implementation**:
    -   `jsCode`:
        ```javascript
         const input = $json.DESCRIPTION;

        // Step 1: Remove markdown code block wrappers (```json and ```)
        const cleanedJson = input
          .replace(/```json\s*/g, '')
          .replace(/```/g, '')
          .trim();

        let parsed;
        try {
          parsed = JSON.parse(cleanedJson);
        } catch (e) {
          throw new Error("Invalid JSON structure in DESCRIPTION");
        }

        // Step 2: Flatten all nested array values into one array
        const allValues = Object.values(parsed).flat();

        // Step 3: Clean and deduplicate
        const uniqueKeywords = [...new Set(
          allValues.map(item => item.toLowerCase().trim())
        )];

        return [
          {
            json: {
              descriptionwords: uniqueKeywords
            }
          }
        ];
        ```
    -   This code removes formatting, extracts the keywords, normalizes them, and outputs them as `descriptionwords`.

### 🔹 Edit Fields4 📝

-   **Function**: Extracts the Gemini's resume keywords response from the API response and sets it to `DESCRIPTION`.
-   **Implementation**:
    -   `assignments`: Sets the `DESCRIPTION` field to `{{ $json.body.candidates[0].content.parts[0].text }}`.

### 🔹 Code3 ⚙️

-   **Function**: Parses the resume keywords text and extracts keywords from a specified `Keywords` section.
-   **Implementation**:
    -   `jsCode`:
        ```javascript
        const input = $json.DESCRIPTION || '';

        const lower = input.toLowerCase();
        const keywordIndex = lower.indexOf('keywords');

        if (keywordIndex === -1) {
          throw new Error("No 'Keywords' section found.");
        }

        // Get the text after 'keywords'
        const afterKeywords = input.slice(keywordIndex + 'keywords'.length);

        // Remove markdown formatting, extra newlines, and special characters except commas
        const cleanedText = afterKeywords
          .replace(/[\*\n\r]/g, ' ')        // remove *, newlines
          .replace(/[^\\w\\s,]/g, '')         // remove all symbols except comma
          .replace(/\s+/g, ' ')             // normalize whitespace
          .trim();

        // Split by commas, trim individual items, and remove duplicates
        const uniqueKeywords = [...new Set(
          cleanedText
            .split(',')
            .map(k => k.trim().toLowerCase())
            .filter(k => k) // remove empty
        )];

        return [
          {
            json: {
              resumewords: uniqueKeywords
            }
          }
        ];
        ```
    -   This code finds and extracts the keywords, cleans them, normalizes them, and outputs them as `resumewords`.

### 🔹 Code4 ⚙️

-   **Function**: Compares the extracted keywords from the resume and job description, calculates a match percentage, and generates a report.
-   **Implementation**:
    -   `jsCode`:
        ```javascript
        // Get both keyword arrays
        const resumeKeywords = $input.first().json.resumewords;
        const descriptionKeywords = $('Code5').first().json.descriptionwords;

        // Normalize for comparison (lowercase and trimmed)
        const normalizedResume = resumeKeywords.map(k => k.toLowerCase().trim());
        const normalizedDescription = descriptionKeywords.map(k => k.toLowerCase().trim());

        // Check which description keywords are present in the resume
        const comparisonResults = normalizedDescription.map((descKeyword, index) => {
          const isPresent = normalizedResume.includes(descKeyword);
          return {
            keyword: descriptionKeywords[index], // original case
            presentInResume: isPresent
          };
        });

        // Prepare summary
        const matched = comparisonResults.filter(item => item.presentInResume);
        const missing = comparisonResults.filter(item => !item.presentInResume);
        const matchPercentage = Math.round((matched.length / descriptionKeywords.length) * 100);

        // Rating
        let rating = '';
        if (matchPercentage >= 80) rating = 'Excellent';
        else if (matchPercentage >= 60) rating = 'Good';
        else if (matchPercentage >= 40) rating = 'Average';
        else rating = 'Poor';

        return [
          {
            json: {
              matchPercentage: matchPercentage + '%',
              matchRating: rating,
              totalKeywordsInJobDescription: descriptionKeywords.length,
              matchedKeywordsCount: matched.length,
              unmatchedKeywordsCount: missing.length,
              matchedKeywords: matched.map(k => k.keyword),
              missingKeywords: missing.map(k => k.keyword),
              fullComparisonReport: comparisonResults
            }
          }
        ];
        ```
    -   This node performs the core comparison logic and produces a comprehensive report.

### 🔹 Send a text message3 💬

-   **Function**: Sends the match report to the user via Telegram.
-   **Implementation**:
    -   `chatId`: `7490692228` (replace with the desired Telegram chat ID).
    -   `text`:
        ```text
        Match Report ✅

        📊 Match Percentage: {{$json.matchPercentage}}

        ⭐ Match Rating: {{$json.matchRating}}

        🧠 Total Keywords in JD: {{$json.totalKeywordsInJobDescription}}

        ✅ Matched: {{$json.matchedKeywordsCount}}

        ❌ Unmatched: {{$json.unmatchedKeywordsCount}}

        ✅ Matched Keywords: {{ $json.matchedKeywords.join(', ') }}

        ❌ Missing Keywords: {{ $json.missingKeywords.join(', ') }}
        ```
    -   This formats the match report and sends it to the specified Telegram chat.
    -   Credentials: `telegramApi` (uses the same Telegram Bot API token).

## 🧪 Testing & Execution

1.  Import the provided JSON into n8n via the "Import Workflow" option.
2.  Ensure that all node connections are valid.
3.  Configure the `telegramApi` credential with a valid Telegram Bot API token.
4.  Configure the Mistral AI OCR API credentials with a valid API token in the "which ocr text reader get text from image" node.
5.  Configure the Google Gemini 2.0 Flash API credentials with valid API key in the "HTTP Request" and "HTTP Request1" nodes.
6.  Replace the sample `chatId` in the "Send a text message3" node with your desired Telegram chat ID.
7.  Activate the workflow.
8.  Send a message containing a resume image or PDF to your Telegram bot.
9.  Review the execution logs for any errors.
10. Check the output of each node to ensure data is flowing as expected.
11. Verify that the match report is sent to your Telegram chat.

## 💡 Notes for Developers

-   **Environment Variables:** The workflow relies on environment variables or n8n credentials for storing sensitive information like API keys and Telegram Bot Tokens. Ensure these are properly configured.
-   **Error Handling:** The `neverError: true` option is set on the "which ocr text reader get text from image", "HTTP Request", and "HTTP Request1" nodes. This allows the workflow to continue execution even if these nodes encounter errors. However, it is crucial to monitor the execution logs for such errors and implement proper error handling to address them.
-   **Scalability:** For handling a large volume of requests, consider implementing queueing mechanisms and scaling the n8n instance horizontally.
-   **Extensibility:** The workflow can be extended to support different job descriptions by modifying the "Edit Fields2" node. Also, the keyword extraction logic can be enhanced by incorporating more sophisticated NLP techniques.

```
