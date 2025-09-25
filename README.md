## Blog Artifacts

This repository stores supporting artifacts for my blog posts. These include code snippets, scripts, and diagrams referenced in the posts.

### 📂 Repository Structure
```
.
├── diagrams/   # Diagrams and visual assets (PNG, SVG, etc.)
└── code/       # Code snippets, scripts, or diagram source code
```

### 🏷️ Naming Convention

- All artifacts are named using a short slug derived from the blog post title.
- This ensures each artifact can be traced back to its corresponding blog post.
- The same slug is used across both diagrams/ and code/.

#### Example:
```
diagrams/
  ecs-automation-arch.png
code/
  ecs-automation-snippets.py
  ecs-automation-diagram-source.puml
```
Here, `ecs-automation` is the slug for the blog post "Automating ECS Deployments".

### 🔗 Usage

- Each blog post will reference its slugged artifacts from this repository.
- To explore or reuse code and diagrams:

```
git clone https://github.com/cybergavin/blogs.git
```
- Look in the `diagrams/` or `code/` folders using the slug for the relevant post.

### 📝 Notes

- Some code files may include inline comments or requirements for reproduction.
