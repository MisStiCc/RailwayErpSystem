# Создайте директорию если нет
mkdir -p .github/workflows

# Создайте ci.yml
cat > .github/workflows/ci.yml << 'EOF'
name: CI/CD Pipeline
on:
  push:
    branches: [ main, develop, feature/** ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    env:
      TELEGRAM_TOKEN: ${{ secrets.TELEGRAM_TOKEN }}
      TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID }}
    
    steps:
    - name: Send Telegram Start
      run: |
        curl -s -X POST "https://api.telegram.org/bot$TELEGRAM_TOKEN/sendMessage" \
          -d "chat_id=$TELEGRAM_CHAT_ID" \
          -d "text=🚀 GitHub Actions: Build Started\n\nRepository: ${{ github.repository }}\nBranch: ${{ github.ref_name }}\nAuthor: ${{ github.actor }}" \
          -d "parse_mode=Markdown"
    
    - uses: actions/checkout@v4
    
    - name: Setup Java
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'
    
    - name: Build with Maven
      run: |
        chmod +x mvnw
        ./mvnw clean compile -DskipTests
      
    - name: Send Telegram Result
      if: always()
      run: |
        if [ "${{ job.status }}" == "success" ]; then
          EMOJI="✅"
          STATUS="SUCCESS"
        else
          EMOJI="❌"
          STATUS="FAILED"
        fi
        
        curl -s -X POST "https://api.telegram.org/bot$TELEGRAM_TOKEN/sendMessage" \
          -d "chat_id=$TELEGRAM_CHAT_ID" \
          -d "text=$EMOJI Build Complete\n\nStatus: $STATUS\nBranch: ${{ github.ref_name }}\nView Logs: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}" \
          -d "parse_mode=Markdown"
EOF
