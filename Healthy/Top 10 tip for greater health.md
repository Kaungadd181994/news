name: Generate HTML List

on:
  push:
    branches:
      - main  # run on pushes to main branch

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout repo
      uses: actions/checkout@v3

    - name: Find all HTML files recursively
      run: |
        # Initialize JSON array
        echo "[" > files.json
        first=true
        # Find all .html files recursively excluding index.html
        find . -type f -name "*.html" ! -name "index.html" | sed 's|^\./||' | while read file; do
          if [ "$first" = true ]; then
            first=false
          else
            echo "," >> files.json
          fi
          echo "  \"$file\"" >> files.json
        done
        echo "]" >> files.json

    - name: Commit JSON
      run: |
        git config --local user.email "action@github.com"
        git config --local user.name "GitHub Action"
        git add files.json
        git diff --quiet || git commit -m "Update files.json"
        git push