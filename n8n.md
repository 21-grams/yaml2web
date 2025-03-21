To create a workflow in n8n that loads Markdown files from your GitHub repository, converts them to static HTML files with CSS styling, and saves them back into an "html" folder within the same repository, follow these steps. This solution leverages n8n's nodes to automate the process, interacting with GitHub's API, performing the Markdown-to-HTML conversion, and applying styling.

### Prerequisites
- **n8n Installed**: Ensure you have n8n set up and running.
- **GitHub Personal Access Token**: Generate a token with `repo` scope permissions to allow reading and writing to your repository. You can create this in GitHub under Settings > Developer settings > Personal access tokens.

### Workflow Overview
1. **Trigger**: Start the workflow on a schedule or manually.
2. **List Markdown Files**: Fetch all files from the repository and filter for `.md` files.
3. **Fetch CSS**: Get a CSS file for styling the HTML.
4. **Process Files**: For each Markdown file:
   - Retrieve its content.
   - Convert it to HTML with styling.
   - Save the HTML file to the "html" folder.

### Step-by-Step Instructions

#### 1. Set Up GitHub Credentials
- In n8n, go to **Credentials** > **Add Credential** > Select "GitHub".
- Enter your personal access token and save it. This will be used to authenticate API requests to your GitHub repository.

#### 2. Trigger the Workflow
- **Node**: **Schedule**
  - **Settings**: Configure it to run periodically (e.g., every day at a specific time) or use a **Manual** trigger if you prefer to run it on demand.
  - **Purpose**: Initiates the workflow.

#### 3. List All Markdown Files
- **Node**: **HTTP Request**
  - **Method**: GET
  - **URL**: `https://api.github.com/repos/{owner}/{repo}/git/trees/{branch}?recursive=1`
    - Replace `{owner}` with your GitHub username or organization.
    - Replace `{repo}` with your repository name.
    - Replace `{branch}` with your branch name (e.g., `main` or `master`).
  - **Authentication**: Select your GitHub credential.
  - **Name**: "List Files" (optional, for clarity).
  - **Purpose**: Retrieves a recursive list of all files in the repository using GitHub's "git/trees" endpoint.

#### 4. Filter for Markdown Files
- **Node**: **Function**
  - **Code**:
    ```javascript
    const tree = $node["List Files"].json["tree"];
    const mdFiles = tree.filter(item => item.type === "blob" && item.path.endsWith(".md"));
    return mdFiles.map(item => ({ json: { path: item.path } }));
    ```
  - **Name**: "Filter Markdown Files" (optional).
  - **Purpose**: Processes the tree data to extract paths of files ending with `.md`, outputting them as a list for the loop.

#### 5. Fetch CSS for Styling
- **Node**: **HTTP Request**
  - **Method**: GET
  - **URL**: `https://cdn.jsdelivr.net/gh/sindresorhus/github-markdown-css/github-markdown.css`
  - **Name**: "Fetch CSS" (optional).
  - **Purpose**: Downloads the GitHub Markdown CSS to style the HTML files similarly to GitHub's Markdown rendering.

#### 6. Loop Over Each Markdown File
- **Node**: **Loop Over Items**
  - **Input**: Connect from the "Filter Markdown Files" node.
  - **Purpose**: Iterates over each Markdown file path.

**Inside the Loop:**

a. **Get Markdown File Content**
- **Node**: **GitHub**
  - **Resource**: File
  - **Operation**: Get
  - **File Path**: `{{$item("path")}}`
    - This expression dynamically uses the current file path from the loop.
  - **Authentication**: Use your GitHub credential.
  - **Name**: "Get Markdown Content" (optional).
  - **Purpose**: Fetches the content of the current Markdown file.

b. **Convert Markdown to HTML with Styling**
- **Node**: **Function**
  - **External Libraries**: Add `showdown` (n8n allows importing this library; check the "Resources" tab in the Function node).
  - **Code**:
    ```javascript
    const showdown = require('showdown');
    const converter = new showdown.Converter();
    const mdContent = $node["Get Markdown Content"].json["content"];
    const htmlContent = converter.makeHtml(mdContent);
    const css = $node["Fetch CSS"].json["body"];
    const filePath = $item("path");
    const title = filePath.split('/').pop().replace('.md', '');
    const fullHtml = `
    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="UTF-8">
      <title>${title}</title>
      <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.5.1/styles/default.min.css">
      <style>
        ${css}
      </style>
    </head>
    <body>
      <div class="markdown-body">
        ${htmlContent}
      </div>
      <script src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.5.1/highlight.min.js"></script>
      <script>hljs.highlightAll();</script>
    </body>
    </html>
    `;
    const htmlPath = 'html/' + filePath.replace('.md', '.html');
    return { json: { html: fullHtml, htmlPath: htmlPath } };
    ```
  - **Name**: "Convert to HTML" (optional).
  - **Purpose**: Converts the Markdown to HTML using Showdown, wraps it in a full HTML structure with the fetched CSS, adds Highlight.js for syntax highlighting, and computes the output file path (e.g., `html/folder/file.html`).

c. **Save the HTML File**
- **Node**: **GitHub**
  - **Resource**: File
  - **Operation**: Create
  - **File Path**: `{{$item("htmlPath")}}`
  - **Content**: `{{$item("html")}}`
  - **Update if Exists**: Enable this option (set to `true`).
  - **Commit Message**: "Update HTML for {{filePath}}" (optional).
  - **Authentication**: Use your GitHub credential.
  - **Name**: "Save HTML File" (optional).
  - **Purpose**: Creates or updates the HTML file in the "html" folder of the repository, preserving the directory structure.

### Final Workflow Structure
```
Schedule
  ↓
HTTP Request (List Files)
  ↓
Function (Filter Markdown Files)
  ↓
HTTP Request (Fetch CSS)
  ↓
Loop Over Items
  ├── GitHub (Get Markdown Content)
  ├── Function (Convert to HTML)
  └── GitHub (Save HTML File)
```

### Notes
- **Styling**: The workflow uses `github-markdown-css` for GitHub-like styling and Highlight.js for syntax highlighting of code blocks. These are linked via CDNs, making the HTML files lightweight but dependent on internet access when viewed. For fully self-contained files, you could inline the CSS and JS, though this increases file size.
- **Customization**: Modify the CSS URL or HTML template in the "Convert to HTML" Function node to use a different style or structure.
- **Error Handling**: For production use, consider adding error handling (e.g., an IF node to check for failed API calls) or a notification node (e.g., email or Slack) to report issues.
- **Directory Creation**: GitHub automatically creates directories like "html" or its subfolders when files are saved to new paths.

### Result
When executed, this workflow will:
- Scan your repository for all `.md` files.
- Convert each to a styled HTML file with syntax highlighting.
- Save them under the "html" folder (e.g., `docs/guide.md` becomes `html/docs/guide.html`).
- Update existing HTML files if they already exist.

You can test the workflow by running it manually first, checking the output in your repository, and then scheduling it as needed. This provides an automated way to maintain static HTML versions of your Markdown content directly within your GitHub repository.
