name: Latest Substack Posts Workflow
on:
  schedule:
    - cron: '0 0 * * *'
  workflow_dispatch:

jobs:
  update-readme-with-blog:
    name: Update this repo's README with latest newsletter posts
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'

      - name: Install dependencies
        run: pip install requests

      - name: Fetch latest Substack posts
        run: |
          python - <<'EOF'
          import re
          import xml.etree.ElementTree as ET

          import requests

          feed_url = "https://ruleofinnovation.substack.com/feed"
          headers = {
              "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36"
          }

          response = requests.get(feed_url, headers=headers, timeout=20)
          response.raise_for_status()

          root = ET.fromstring(response.content)
          items = root.findall(".//item")
          if not items:
              items = root.findall(".//{http://www.w3.org/2005/Atom}entry")

          posts = []
          for item in items[:5]:
              if item.tag.endswith("item"):
                  title = (item.findtext("title") or "").strip()
                  link = (item.findtext("link") or "").strip()
              else:
                  title = (item.findtext("{http://www.w3.org/2005/Atom}title") or "").strip()
                  link_tag = item.find("{http://www.w3.org/2005/Atom}link")
                  link = (link_tag.get("href") if link_tag is not None else "").strip()

              if title and link:
                  posts.append(f"- 📰 [{title}]({link})")

          if not posts:
              raise RuntimeError("No Substack posts were returned by the feed.")

          with open("README.md", "r", encoding="utf-8") as f:
              readme = f.read()

          pattern = r"(<!-- BLOG-POST-LIST:START -->)(.*?)(<!-- BLOG-POST-LIST:END -->)"
          replacement = "\\1\n" + "\n".join(posts) + "\n\\3"
          updated_readme = re.sub(pattern, replacement, readme, flags=re.DOTALL)

          if updated_readme == readme:
              raise RuntimeError("README markers were not found. Add the Substack block markers to the file.")

          with open("README.md", "w", encoding="utf-8") as f:
              f.write(updated_readme)

          print("Latest Substack posts synced successfully.")
          EOF

      - name: Commit and Push Changes
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add README.md
          git diff-index --quiet HEAD || git commit -m "docs: sync latest newsletter posts"
          git push origin main
