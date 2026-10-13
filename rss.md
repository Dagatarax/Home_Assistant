Creates template for x number of news with details saved in attributes, triggered from Feedreader integration (native, not HACS). On Voice Assistant conversation trigger reads the news aloud - titles and details. Also it shows them on Dashboard in custom card (no HACS). No LLM/Ai needed.

Google Gemini (free tier) helped with this. Feel free to ask it for customization. If it nicked your ideas, I am sorry.

# **Prerequisites:**

* [ ] Voice Assistant (Voice PE) device with TTS and SST pipeline (Dashboard only for less cool experience)


* [ ] Split `configuration.yaml`, i.e. containing:
  `template: !include_dir_merge_list ./templates/`


* [ ] Integration `Feedreader`

with Services for EuroNews and Hacker News

<<https://news.ycombinator.com/rss>>

<<https://www.euronews.com/rss>>

# Settings:

## Templates:

Create these files for each RSS service:

`/homeassistant/templates/euronews.yaml`

```
- trigger:
    - trigger: event
      event_type: feedreader
      event_data:
        feed_url: "https://www.euronews.com/rss"
  sensor:
    - name: "Euro News Feed"
      unique_id: euro_news_feed
      icon: mdi:newspaper
      state: >
        {% set existing = (this.attributes.get('entries', []) if this and this.attributes else []) or [] %}
        {% set d = trigger.event.data if trigger is defined and trigger.event is defined else none %}
        {% if d %}
          {% set new_entry = {
            'title': d.get('title', 'No Title'),
            'link': d.get('link', ''),
            'description': d.get('description', ''),
            'published': d.get('published', '')
          } %}
          {{ ([new_entry] + existing)[:10] | count }}
        {% else %}
          {{ existing | count }}
        {% endif %}
      attributes:
        entries: >
          {% set existing = (this.attributes.get('entries', []) if this and this.attributes else []) or [] %}
          {% set d = trigger.event.data if trigger is defined and trigger.event is defined else none %}
          {% if d %}
            {% set new_entry = {
              'title': d.get('title', 'No Title'),
              'link': d.get('link', ''),
              'description': d.get('description', ''),
              'published': d.get('published', '')
            } %}
            {{ ([new_entry] + existing)[:10] }}
          {% else %}
            {{ existing }}
          {% endif %}
```

Pulls `10` headlines with details.

`/homeassistant/templates/hackernews.yaml`

```
- trigger:
    - trigger: state
      entity_id: event.hacker_news
      not_from:
        - "unavailable"
        - "unknown"
  sensor:
    - name: "Hacker News Feed"
      unique_id: hacker_news_feed
      icon: mdi:code-tags
      state: >
        {% set existing = (this.attributes.get('entries', []) if this and this.attributes else []) %}
        {% set d = trigger.to_state.attributes if trigger is defined and trigger.to_state is defined else none %}
        {% if d and d.get('title') %}
          {% set new_entry = {
            'title': d.get('title', 'No Title'),
            'link': d.get('link', ''),
            'description': d.get('description', ''),
            'published': d.get('published', '')
          } %}
          {{ ([new_entry] + existing)[:10] | count }}
        {% else %}
          {{ existing | count }}
        {% endif %}
      attributes:
        entries: >
          {% set existing = (this.attributes.get('entries', []) if this and this.attributes else []) %}
          {% set d = trigger.to_state.attributes if trigger is defined and trigger.to_state is defined else none %}
          {% if d and d.get('title') %}
            {% set new_entry = {
              'title': d.get('title', 'No Title'),
              'link': d.get('link', ''),
              'description': d.get('description', ''),
              'published': d.get('published', '')
            } %}
            {{ ([new_entry] + existing)[:10] }}
          {% else %}
            {{ existing }}
          {% endif %}
```

Pulls `10` headlines with details (Hacker news is sparse with details, tho).

**Check configuration** `/config/tools/yaml` and reload | restart

## Script:

```
sequence:
  - action: assist_satellite.announce
    metadata: {}
    target:
      device_id: '{{ target_satellite }}'
    data:
      preannounce: false
      message: >-
        {% set entries = state_attr('sensor.euro_news_feed', 'entries') |
        default([]) %} {% if entries | length > 0 %}
          Here are the top news headlines and details from Euro News,,,,,,,,,,,,,,,,,    
          {% for entry in entries[:10] %}
            {{ entry.title }}: {{ entry.description | striptags | truncate(200) }} NEXT,,,,,,,,,,,,,,,,,,   
          {% endfor %}
        {% else %}
          There are currently no news updates available.
        {% endif %} 
          End of briefing, sir
  - action: assist_satellite.announce
    metadata: {}
    target:
      device_id: '{{ target_satellite }}'
    data:
      preannounce: false
      message: >-
        {% set entries = state_attr('sensor.hacker_news_feed', 'entries') |
        default([]) %} {% if entries | length > 0 %}
          Here are the top news headlines and details from Hacker News,,,,,,,,,,,,,,,,,    
          {% for entry in entries[:10] %}
            {{ entry.title }}: NEXT,,,,,,,,,,,,,,,,,,   
          {% endfor %}
        {% else %}
          There are currently no news updates available.
        {% endif %} 
          End of briefing, sir
alias: Read_News
description: ''
```

Piper TTS is somewhat fast reader. `,,,,,,,,,,,,,,,,,,` is to slow it down a bit.

`:10` reads just the 10 entries, truncated at 200 characters (some bulletins can be pretty large otherwise).

## Automation:

```
alias: Voice_Trigger-Read_News
description: ''
triggers:
  - trigger: conversation
    command:
      - Read news
      - News
      - Headlines
      - What's new
conditions: []
actions:
  - action: script.read_news
    metadata: {}
    data:
      target_satellite: '{{ trigger.device_id }}'
mode: single
```

This passes argument to the script, so the script answers on the device triggered (handy if you have two or more).

## Dashboard:

### **Markdown card configuration**

Best set as **Footer** of max width.

```
<div style="font-size: 32px;">
  {% set entries = state_attr('sensor.euro_news_feed', 'entries') | default([]) %}
  {% if entries | length > 0 %}
    {% set ns = namespace(news=[]) %}
    {% for entry in entries %}
      {% set item = entry.title ~ ' — ' ~ (entry.description | striptags | truncate(100)) %}
      {% set ns.news = ns.news + [item] %}
    {% endfor %}
    {{ ns.news | join('     •     ') }}
  {% else %}
    Waiting for real news feed updates...
  {% endif %}
</div>
```

Only Euronews feed here, tho.

# ENJOY
