# 🍳 Odoo Code Snippets & Cookbook

Welcome to the unofficial **Odoo Cookbook**! This repository is a curated collection of ready-to-use Odoo code snippets, templates, and architectural best practices. It is designed to save developers and technical consultants hours of searching through Odoo's core source code.

All snippets are formatted for easy copy-pasting, focus on modern Odoo versions (18, 19, and 20), and follow the strict coding guidelines established by the Odoo Community Association (OCA).

## 📚 What's Inside

The documentation is divided into clear architectural layers:

1. **ORM & Models:** Computed fields (`@api.depends`), validations (`@api.constrains`), CRUD overrides, and efficient recordset operations.

2. **Views & UI (XML):** Safe form inheritance (avoiding `replace`), dynamic lists, search filters, and Kanban boards.

3. **Chatter & Mail:** Activity tracking, silent logging (`mail.mt_note`), and automated template-based emails.

4. **Wizards:** Transient models (`models.TransientModel`) for mass updates and multi-step interactive modals.

5. **Actions & Cron:** Scheduled background jobs (`ir.cron`), server actions, and contextual batch processes.

6. **OWL & Frontend:** Modern JavaScript UI components, custom field widgets, and patching existing hooks/lifecycles.

7. **Migration & Versions:** Quick reference guides for upgrading syntax (e.g., the death of `attrs`, the transition from `<tree>` to `<list>`).

8. **Best Practices & Pitfalls:** Common mistakes (N+1 queries, Expected Singleton, Context pollution) and how to avoid them for optimized, bug-free code.

## 🗂️ Project Structure

The documentation is built using [MkDocs](https://www.mkdocs.org/) with the [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) theme.

```
odoo-cookbook/
├── mkdocs.yml              # MkDocs configuration and navigation menu
├── .github/workflows/      # GitHub Actions for automated GH Pages deployment
└── docs/                   # Markdown files containing the cookbook recipes
    ├── index.md
    ├── 01_orm_models.md
    ├── 02_views_ui.md
    ├── ...
    └── 08_best_practices.md

```

## 🛠️ Local Installation & Development

If you want to run this documentation site locally to preview your changes, we recommend using [`uv`](https://github.com/astral-sh/uv) for lightning-fast Python environment management.

1. **Clone the repository:**

   ```
   git clone https://github.com/Lucgril/odoo-cookbook.git
   cd odoo-cookbook
   
   ```

2. **Create a virtual environment and install dependencies:**
   Using `uv`, you can install MkDocs and the Material theme directly:

   ```
   uv venv
   uv pip install mkdocs-material
   
   ```

3. **Start the local server:**

   ```
   uv run mkdocs serve
   
   ```

   *Open your browser and navigate to `http://127.0.0.1:8000` to see the live site. The page will auto-reload when you edit any `.md` file.*

## 🤝 Contributing

Contributions are highly welcome! If you have a brilliant snippet, a missing workflow, or a correction:

1. Fork this repository.

2. Create a new branch (`git checkout -b feature/new-snippet`).

3. Add your Markdown content to the appropriate file in the `docs/` folder. Make sure to use the ```` ```python ```` or ```` ```xml ```` blocks for code.

4. Commit your changes (`git commit -m 'feat: add snippet for dynamic domains'`).

5. Push to the branch (`git push origin feature/new-snippet`).

6. Open a Pull Request.

## 📄 License

This project is licensed under the **MIT License**. You are free to copy, modify, and use these snippets in both your open-source and commercial/private Odoo projects without restrictions. See the `LICENSE` file for more details.