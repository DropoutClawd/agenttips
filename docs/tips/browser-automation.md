# Browser Automation: Teaching Your Agent to Navigate the Web

Browser automation transforms your agent from a text processor into a web-native assistant. Fill forms, extract data, click buttons, and navigate complex web apps - all programmatically.

## The Browser Automation Stack

| Tool | Best For | Learning Curve | Speed |
|------|----------|----------------|-------|
| Playwright | Full automation, testing | Medium | Fast |
| Puppeteer | Chrome-specific, CDP | Medium | Fast |
| Selenium | Legacy, multi-browser | High | Slow |
| Browser Use | LLM-native navigation | Low | Medium |

## Strategy 1: Playwright Fundamentals

```python
import asyncio
from playwright.async_api import async_playwright, Page, Browser
from typing import Optional, Dict, Any, List
import json

class BrowserAgent:
    """Browser automation agent using Playwright"""
    
    def __init__(self, headless: bool = True):
        self.headless = headless
        self.browser: Optional[Browser] = None
        self.page: Optional[Page] = None
        self.playwright = None
    
    async def start(self):
        """Initialize browser"""
        self.playwright = await async_playwright().start()
        self.browser = await self.playwright.chromium.launch(
            headless=self.headless,
            args=[
                '--disable-blink-features=AutomationControlled',
                '--no-sandbox',
                '--disable-dev-shm-usage'
            ]
        )
        
        # Create context with realistic settings
        context = await self.browser.new_context(
            viewport={'width': 1920, 'height': 1080},
            user_agent='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
            locale='en-US',
            timezone_id='America/New_York'
        )
        
        self.page = await context.new_page()
        
        # Block unnecessary resources for speed
        await self.page.route("**/*.{png,jpg,jpeg,gif,webp}", lambda route: route.abort())
    
    async def stop(self):
        """Clean up"""
        if self.browser:
            await self.browser.close()
        if self.playwright:
            await self.playwright.stop()
    
    async def navigate(self, url: str, wait_for: str = "networkidle"):
        """Navigate to URL and wait for page load"""
        await self.page.goto(url, wait_until=wait_for)
        await self.page.wait_for_load_state("domcontentloaded")
    
    async def extract_text(self, selector: str) -> str:
        """Extract text from element"""
        element = await self.page.query_selector(selector)
        if element:
            return await element.inner_text()
        return ""
    
    async def extract_all_text(self, selector: str) -> List[str]:
        """Extract text from all matching elements"""
        elements = await self.page.query_selector_all(selector)
        return [await el.inner_text() for el in elements]
    
    async def fill_form(self, fields: Dict[str, str]):
        """Fill form fields"""
        for selector, value in fields.items():
            await self.page.fill(selector, value)
    
    async def click(self, selector: str, wait_after: int = 1000):
        """Click element and wait"""
        await self.page.click(selector)
        await self.page.wait_for_timeout(wait_after)
    
    async def screenshot(self, path: str, full_page: bool = False):
        """Take screenshot"""
        await self.page.screenshot(path=path, full_page=full_page)
    
    async def get_page_content(self) -> str:
        """Get full page HTML"""
        return await self.page.content()
    
    async def evaluate(self, script: str) -> Any:
        """Execute JavaScript in page context"""
        return await self.page.evaluate(script)
    
    async def wait_for_selector(self, selector: str, timeout: int = 30000):
        """Wait for element to appear"""
        await self.page.wait_for_selector(selector, timeout=timeout)

# Usage example
async def scrape_hacker_news():
    agent = BrowserAgent(headless=True)
    await agent.start()
    
    try:
        await agent.navigate("https://news.ycombinator.com")
        
        # Extract all story titles
        titles = await agent.extract_all_text(".titleline > a")
        
        # Extract scores
        scores = await agent.extract_all_text(".score")
        
        # Combine into structured data
        stories = [
            {"title": title, "score": score}
            for title, score in zip(titles, scores)
        ]
        
        return stories[:10]  # Top 10
        
    finally:
        await agent.stop()

# Run
stories = asyncio.run(scrape_hacker_news())
for story in stories:
    print(f"{story['score']}: {story['title']}")
```

## Strategy 2: LLM-Guided Navigation

```python
from typing import List, Dict, Any
import json
import re

class LLMBrowserAgent:
    """Use LLM to understand and navigate web pages"""
    
    def __init__(self, browser_agent: BrowserAgent, llm_client):
        self.browser = browser_agent
        self.llm = llm_client
    
    async def get_page_structure(self) -> str:
        """Get simplified page structure for LLM"""
        structure = await self.browser.evaluate("""
            () => {
                function getStructure(element, depth = 0) {
                    if (depth > 3) return '';
                    if (!element || !element.tagName) return '';
                    
                    const tag = element.tagName.toLowerCase();
                    const id = element.id ? `#${element.id}` : '';
                    const classes = element.className ? 
                        `.${element.className.split(' ').slice(0, 2).join('.')}` : '';
                    const text = element.innerText?.slice(0, 50) || '';
                    
                    // Focus on interactive elements
                    const interactive = ['a', 'button', 'input', 'select', 'textarea', 'form'];
                    
                    let result = '';
                    if (interactive.includes(tag) || id || classes) {
                        result = `${'  '.repeat(depth)}<${tag}${id}${classes}>${text}</${tag}>\\n`;
                    }
                    
                    for (const child of element.children) {
                        result += getStructure(child, depth + 1);
                    }
                    
                    return result;
                }
                return getStructure(document.body);
            }
        """)
        return structure
    
    async def find_element_for_action(self, action: str) -> str:
        """Use LLM to find the right element for an action"""
        structure = await self.get_page_structure()
        
        prompt = f"""Given this page structure:
{structure}

Find the CSS selector for: {action}

Respond with ONLY the CSS selector, nothing else.
Example response: button.submit-btn
"""
        
        response = await self.llm.messages.create(
            model="claude-3-haiku-20240307",
            messages=[{"role": "user", "content": prompt}],
            max_tokens=100
        )
        
        return response.content[0].text.strip()
    
    async def execute_task(self, task: str) -> Dict[str, Any]:
        """Execute a natural language task"""
        
        # Get current page state
        url = self.browser.page.url
        title = await self.browser.page.title()
        structure = await self.get_page_structure()
        
        # Ask LLM for action plan
        prompt = f"""You are a browser automation agent.

Current page: {url}
Title: {title}

Page structure:
{structure[:3000]}

Task: {task}

What actions should I take? Respond with a JSON array of actions:
[
    {{"action": "click", "selector": "...", "description": "..."}},
    {{"action": "fill", "selector": "...", "value": "...", "description": "..."}},
    {{"action": "navigate", "url": "...", "description": "..."}},
    {{"action": "extract", "selector": "...", "description": "..."}},
    {{"action": "wait", "selector": "...", "description": "..."}}
]

Respond with ONLY the JSON array."""
        
        response = await self.llm.messages.create(
            model="claude-3-5-sonnet-20241022",
            messages=[{"role": "user", "content": prompt}],
            max_tokens=1000
        )
        
        # Parse actions
        text = response.content[0].text
        json_match = re.search(r'\[[\s\S]*\]', text)
        if not json_match:
            raise ValueError("Could not parse action plan")
        
        actions = json.loads(json_match.group())
        results = []
        
        # Execute each action
        for action in actions:
            try:
                result = await self._execute_action(action)
                results.append({"action": action, "result": result, "success": True})
            except Exception as e:
                results.append({"action": action, "error": str(e), "success": False})
        
        return {"actions": results}
    
    async def _execute_action(self, action: Dict) -> Any:
        """Execute a single action"""
        action_type = action["action"]
        
        if action_type == "click":
            await self.browser.click(action["selector"])
            return "clicked"
        
        elif action_type == "fill":
            await self.browser.page.fill(action["selector"], action["value"])
            return "filled"
        
        elif action_type == "navigate":
            await self.browser.navigate(action["url"])
            return "navigated"
        
        elif action_type == "extract":
            text = await self.browser.extract_text(action["selector"])
            return text
        
        elif action_type == "wait":
            await self.browser.wait_for_selector(action["selector"])
            return "found"
        
        else:
            raise ValueError(f"Unknown action: {action_type}")

# Usage
async def search_and_extract():
    browser = BrowserAgent(headless=False)  # Show browser for debugging
    await browser.start()
    
    agent = LLMBrowserAgent(browser, anthropic_client)
    
    try:
        await browser.navigate("https://google.com")
        
        result = await agent.execute_task(
            "Search for 'best practices for AI agents' and extract the first 5 result titles"
        )
        
        print(json.dumps(result, indent=2))
        
    finally:
        await browser.stop()
```

## Strategy 3: Handling Dynamic Content

```python
class DynamicContentHandler:
    """Handle SPAs and dynamically loaded content"""
    
    def __init__(self, page: Page):
        self.page = page
    
    async def wait_for_content(
        self,
        selector: str,
        timeout: int = 30000,
        poll_interval: int = 100
    ) -> bool:
        """Wait for content to appear with polling"""
        import time
        start = time.time()
        
        while (time.time() - start) * 1000 < timeout:
            element = await self.page.query_selector(selector)
            if element:
                # Check if element has content
                text = await element.inner_text()
                if text.strip():
                    return True
            await self.page.wait_for_timeout(poll_interval)
        
        return False
    
    async def wait_for_network_idle(self, timeout: int = 5000):
        """Wait for network to be idle"""
        await self.page.wait_for_load_state("networkidle", timeout=timeout)
    
    async def scroll_and_load(
        self,
        item_selector: str,
        target_count: int = 50,
        max_scrolls: int = 20
    ) -> List[Any]:
        """Scroll to load lazy-loaded content"""
        items = []
        scroll_count = 0
        
        while len(items) < target_count and scroll_count < max_scrolls:
            # Get current items
            elements = await self.page.query_selector_all(item_selector)
            items = [await el.inner_text() for el in elements]
            
            if len(items) >= target_count:
                break
            
            # Scroll down
            await self.page.evaluate("window.scrollTo(0, document.body.scrollHeight)")
            await self.page.wait_for_timeout(1000)  # Wait for content to load
            
            scroll_count += 1
        
        return items[:target_count]
    
    async def wait_for_spa_navigation(self, expected_url_part: str, timeout: int = 10000):
        """Wait for SPA navigation to complete"""
        await self.page.wait_for_url(f"**/*{expected_url_part}*", timeout=timeout)
        await self.page.wait_for_load_state("networkidle")
    
    async def intercept_api_responses(
        self,
        url_pattern: str,
        callback
    ):
        """Intercept and process API responses"""
        async def handle_response(response):
            if url_pattern in response.url:
                try:
                    data = await response.json()
                    await callback(data)
                except:
                    pass
        
        self.page.on("response", handle_response)
    
    async def wait_for_mutation(
        self,
        selector: str,
        timeout: int = 10000
    ) -> bool:
        """Wait for DOM mutation in element"""
        return await self.page.evaluate(f"""
            async () => {{
                return new Promise((resolve) => {{
                    const target = document.querySelector('{selector}');
                    if (!target) {{
                        resolve(false);
                        return;
                    }}
                    
                    const observer = new MutationObserver(() => {{
                        observer.disconnect();
                        resolve(true);
                    }});
                    
                    observer.observe(target, {{
                        childList: true,
                        subtree: true,
                        characterData: true
                    }});
                    
                    setTimeout(() => {{
                        observer.disconnect();
                        resolve(false);
                    }}, {timeout});
                }});
            }}
        """)

# Usage for infinite scroll
async def scrape_infinite_scroll_page():
    browser = BrowserAgent()
    await browser.start()
    
    handler = DynamicContentHandler(browser.page)
    
    await browser.navigate("https://example.com/infinite-list")
    
    # Scroll and collect items
    items = await handler.scroll_and_load(
        item_selector=".list-item",
        target_count=100,
        max_scrolls=30
    )
    
    print(f"Collected {len(items)} items")
    await browser.stop()
```

## Strategy 4: Anti-Detection Techniques

```python
import random
from typing import List

class StealthBrowser:
    """Browser automation with anti-detection measures"""
    
    USER_AGENTS = [
        'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
        'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
        'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
    ]
    
    @classmethod
    async def create_stealth_context(cls, browser):
        """Create browser context with stealth settings"""
        context = await browser.new_context(
            viewport={'width': random.randint(1200, 1920), 'height': random.randint(800, 1080)},
            user_agent=random.choice(cls.USER_AGENTS),
            locale=random.choice(['en-US', 'en-GB', 'en-AU']),
            timezone_id=random.choice(['America/New_York', 'America/Los_Angeles', 'Europe/London']),
            geolocation={'latitude': 40.7128, 'longitude': -74.0060},
            permissions=['geolocation']
        )
        
        page = await context.new_page()
        
        # Inject stealth scripts
        await page.add_init_script("""
            // Override navigator.webdriver
            Object.defineProperty(navigator, 'webdriver', {
                get: () => undefined
            });
            
            // Override chrome detection
            window.chrome = {
                runtime: {}
            };
            
            // Override permissions
            const originalQuery = window.navigator.permissions.query;
            window.navigator.permissions.query = (parameters) => (
                parameters.name === 'notifications' ?
                    Promise.resolve({ state: Notification.permission }) :
                    originalQuery(parameters)
            );
            
            // Override plugins
            Object.defineProperty(navigator, 'plugins', {
                get: () => [1, 2, 3, 4, 5]
            });
            
            // Override languages
            Object.defineProperty(navigator, 'languages', {
                get: () => ['en-US', 'en']
            });
        """)
        
        return context, page
    
    @staticmethod
    async def human_like_click(page, selector: str):
        """Click with human-like behavior"""
        element = await page.query_selector(selector)
        if not element:
            raise ValueError(f"Element not found: {selector}")
        
        # Get element position
        box = await element.bounding_box()
        if not box:
            raise ValueError("Element not visible")
        
        # Random position within element
        x = box['x'] + random.uniform(5, box['width'] - 5)
        y = box['y'] + random.uniform(5, box['height'] - 5)
        
        # Move mouse with slight curve
        await page.mouse.move(x, y, steps=random.randint(10, 25))
        
        # Random delay before click
        await page.wait_for_timeout(random.randint(50, 200))
        
        # Click
        await page.mouse.click(x, y)
    
    @staticmethod
    async def human_like_type(page, selector: str, text: str):
        """Type with human-like delays"""
        await page.click(selector)
        
        for char in text:
            await page.keyboard.type(char)
            # Variable delay between keystrokes
            await page.wait_for_timeout(random.randint(50, 150))
    
    @staticmethod
    async def random_scroll(page):
        """Perform random scrolling to appear human"""
        scroll_amount = random.randint(100, 500)
        await page.mouse.wheel(0, scroll_amount)
        await page.wait_for_timeout(random.randint(500, 1500))
    
    @staticmethod
    async def add_random_delays(page, min_ms: int = 500, max_ms: int = 2000):
        """Add random delay between actions"""
        await page.wait_for_timeout(random.randint(min_ms, max_ms))

# Usage
async def stealthy_scrape():
    playwright = await async_playwright().start()
    browser = await playwright.chromium.launch(headless=False)
    
    context, page = await StealthBrowser.create_stealth_context(browser)
    
    await page.goto("https://example.com")
    
    # Human-like interactions
    await StealthBrowser.human_like_click(page, "input[name='search']")
    await StealthBrowser.human_like_type(page, "input[name='search']", "AI agents")
    await StealthBrowser.random_scroll(page)
    await StealthBrowser.add_random_delays(page)
    
    await browser.close()
```

## Strategy 5: Parallel Browser Operations

```python
import asyncio
from typing import List, Dict, Callable, Any
from dataclasses import dataclass

@dataclass
class BrowserTask:
    url: str
    extractor: Callable
    name: str = ""

class ParallelBrowserPool:
    """Manage multiple browser instances for parallel scraping"""
    
    def __init__(self, max_browsers: int = 5):
        self.max_browsers = max_browsers
        self.playwright = None
        self.browsers: List[Browser] = []
        self.available: asyncio.Queue = None
    
    async def start(self):
        """Initialize browser pool"""
        self.playwright = await async_playwright().start()
        self.available = asyncio.Queue()
        
        for i in range(self.max_browsers):
            browser = await self.playwright.chromium.launch(headless=True)
            self.browsers.append(browser)
            await self.available.put(browser)
    
    async def stop(self):
        """Clean up all browsers"""
        for browser in self.browsers:
            await browser.close()
        if self.playwright:
            await self.playwright.stop()
    
    async def _process_task(self, task: BrowserTask) -> Dict:
        """Process a single task with a browser from the pool"""
        browser = await self.available.get()
        
        try:
            context = await browser.new_context()
            page = await context.new_page()
            
            await page.goto(task.url, wait_until="networkidle")
            result = await task.extractor(page)
            
            await context.close()
            
            return {
                "name": task.name,
                "url": task.url,
                "result": result,
                "success": True
            }
        
        except Exception as e:
            return {
                "name": task.name,
                "url": task.url,
                "error": str(e),
                "success": False
            }
        
        finally:
            await self.available.put(browser)
    
    async def process_all(self, tasks: List[BrowserTask]) -> List[Dict]:
        """Process all tasks with parallel browsers"""
        return await asyncio.gather(
            *[self._process_task(task) for task in tasks]
        )

# Usage
async def parallel_scrape_example():
    pool = ParallelBrowserPool(max_browsers=3)
    await pool.start()
    
    # Define extraction function
    async def extract_title(page):
        return await page.title()
    
    # Create tasks
    tasks = [
        BrowserTask(url="https://news.ycombinator.com", extractor=extract_title, name="HN"),
        BrowserTask(url="https://reddit.com", extractor=extract_title, name="Reddit"),
        BrowserTask(url="https://github.com", extractor=extract_title, name="GitHub"),
        BrowserTask(url="https://stackoverflow.com", extractor=extract_title, name="SO"),
    ]
    
    results = await pool.process_all(tasks)
    
    for r in results:
        if r["success"]:
            print(f"{r['name']}: {r['result']}")
        else:
            print(f"{r['name']}: ERROR - {r['error']}")
    
    await pool.stop()
```

## Common Pitfalls

### ❌ Not Handling Page Load States
```python
# Don't do this
await page.goto(url)
await page.click(selector)  # Element might not exist yet!

# Do this
await page.goto(url, wait_until="networkidle")
await page.wait_for_selector(selector)
await page.click(selector)
```

### ❌ Hardcoded Waits
```python
# Don't do this
await page.wait_for_timeout(5000)  # Wastes time or not enough

# Do this
await page.wait_for_selector(".loaded-indicator")
```

### ❌ Not Cleaning Up
```python
# Always clean up browsers
try:
    await do_automation()
finally:
    await browser.close()
    await playwright.stop()
```

## Pro Tips

### 1. Use Page Object Pattern
```python
class LoginPage:
    def __init__(self, page):
        self.page = page
        self.email_input = "input[name='email']"
        self.password_input = "input[name='password']"
        self.submit_button = "button[type='submit']"
    
    async def login(self, email: str, password: str):
        await self.page.fill(self.email_input, email)
        await self.page.fill(self.password_input, password)
        await self.page.click(self.submit_button)
        await self.page.wait_for_url("**/dashboard**")
```

### 2. Screenshot on Failure
```python
async def safe_action(page, action_func, screenshot_dir: str = "errors"):
    try:
        return await action_func()
    except Exception as e:
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        await page.screenshot(path=f"{screenshot_dir}/error_{timestamp}.png")
        raise
```

### 3. Session Persistence
```python
# Save session for reuse
storage = await context.storage_state()
with open("session.json", "w") as f:
    json.dump(storage, f)

# Restore session
context = await browser.new_context(storage_state="session.json")
```

## Quick Checklist

- [ ] Wait for proper page load states
- [ ] Handle dynamic content appropriately
- [ ] Implement anti-detection when needed
- [ ] Use parallel browsers for speed
- [ ] Clean up resources properly
- [ ] Screenshot on failures for debugging
- [ ] Use Page Object pattern for maintainability
- [ ] Persist sessions when appropriate
- [ ] Respect robots.txt and rate limits
- [ ] Test with both headless and headed modes

Remember: **Browser automation is powerful but fragile. Build resilient selectors, expect failures, and always have fallback strategies.** The web changes constantly - your automation should handle that gracefully.