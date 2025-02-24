# 宠物智能文案工厂.dify.yaml
version: "1.1"
name: PetProduct_Copy_Generator
description: 实时热点驱动的多平台宠物用品文案生成系统

# ================= 数据源配置 =================
data_sources:
  - type: http
    name: realtime_trends
    config:
      url: "https://api.trends.com/v1/pet/trending?platform=douyin,xhs"
      method: GET
      headers:
        Authorization: "Bearer ${env.TRENDS_API_KEY}"
      schedule: "0 */2 * * *"  # 每2小时更新
      output_processing: |
        (ctx) => ({
          hashtags: ctx.data.top_hashtags.slice(0,5),
          rising_products: ctx.data.rising_products.filter(p => p.category === 'pet')
        })

  - type: database
    name: product_db
    config:
      connection: "${env.PRODUCT_DB_URL}"
      query: "SELECT * FROM products WHERE status = 'active'"

# ================= AI 模型配置 =================
models:
  - name: copy_generator
    type: custom
    config:
      base_url: "https://api.deepseek.com/v1"
      api_key: "${env.DEEPSEEK_API_KEY}"
      parameters:
        temperature: 0.7
        max_tokens: 500
        stop_sequences: ["###"]
    prompt_template: |
      你是宠物用品营销专家，结合以下要素生成文案：
      [平台] {{platform}} 
      [热点] {{hashtags | join(', ')}}
      [产品] {{product.name}} - {{product.features}}
      [风格要求] {{style_guide}}
      
      生成{{platform}}文案要求：
      {% if platform == 'douyin' %}
      - 使用悬念式开头和行动号召
      - 添加话题：{{hashtags[0]}} #养宠必备
      {% elif platform == 'xhs' %}
      - 包含3个emoji和2个使用场景描述
      - 添加#萌宠日常 #好物推荐
      {% endif %}

# ================= 主工作流 =================
workflow:
  start:
    - trigger: 
        type: schedule
        cron: "0 */3 * * *"  # 每3小时自动运行
    - trigger: 
        type: manual
  
  steps:
    - name: fetch_data
      type: parallel
      branches:
        - data_source: realtime_trends
        - data_source: product_db

    - name: match_trends
      type: script
      code: |
        // 匹配产品与热点
        const trendingKeywords = ctx.realtime_trends.hashtags;
        ctx.products = ctx.product_db.filter(product => 
          trendingKeywords.some(kw => product.keywords.includes(kw))
        return { 
          matched_products: ctx.products.slice(0,5) 
        }

    - name: generate_copy
      type: foreach
      input: "${matched_products}"
      steps:
        - name: select_platform
          type: condition
          cases:
            - if: "${product.type in ['饰品','吊牌']}"
              set: platform = "xhs"
            - else:
              set: platform = "douyin"

        - name: get_style_guide
          type: database_query
          config:
            query: "SELECT * FROM style_guides WHERE platform = '${platform}'"
          output: style_guide

        - name: ai_generation
          type: model_inference
          model: copy_generator
          input: 
            product: "${item}"
            platform: "${platform}"
            hashtags: "${realtime_trends.hashtags}"
            style_guide: "${style_guide.content}"

        - name: format_output
          type: script
          code: |
            // 结构化输出
            const copy = ctx.ai_generation.output;
            return {
              platform: ctx.platform,
              product_id: ctx.item.id,
              content: {
                text: copy,
                hashtags: extractHashtags(copy), // 自定义函数提取话题
                meta: {
                  char_count: copy.length,
                  emoji_count: (copy.match(/[\u{1F600}-\u{1F64F}]/gu) || []).length
                }
              }
            }

    - name: quality_check
      type: script
      code: |
        // 质量过滤
        ctx.valid_copies = ctx.generate_copy.filter(item => 
          item.content.meta.char_count >= 50 && 
          item.content.meta.emoji_count >= 2
        )

    - name: publish
      type: action
      config:
        douyin: 
          type: http
          url: "https://api.douyin.com/publish"
          method: POST
          body: "${map(valid_copies, c => ({text: c.content.text}))}"
        xhs:
          type: http
          url: "https://api.xiaohongshu.com/v2/post"
          method: POST
          body: "${valid_copies}"

# ================= 监控与反馈 =================
monitoring:
  - name: performance_tracking
    type: webhook
    config:
      url: "${env.ANALYTICS_WEBHOOK}"
      events: ["publish_success", "publish_failed"]
      data_template: |
        {
          "timestamp": "${timestamp}",
          "product_id": "${product_id}",
          "platform": "${platform}",
          "metrics": {
            "ctr": "${event.data.ctr}", 
            "conversion_rate": "${event.data.conversion}"
          }
        }

  - name: retry_failed
    type: retry
    config:
      max_attempts: 3
      backoff: 1000
      on_failure: "alert('发布失败: ${error.message}')"

# ================= 辅助函数 =================
functions:
  - name: extractHashtags
    code: |
      (text) => {
        const regex = /#(\w+)/g;
        return [...new Set([...text.matchAll(regex)].map(m => m[0]))];
      }
