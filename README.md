import { chromium } from 'playwright';

(async () => {
  let browser;

  try {
    console.log('🚀 Starting browser...');

    browser = await chromium.launch({
      headless: true,
    });

    const page = await browser.newPage();

    // Browser console messages
    page.on('console', (msg) => {
      const type = msg.type().toUpperCase();
      console.log(`[BROWSER ${type}] ${msg.text()}`);
    });

    // JavaScript errors from the page
    page.on('pageerror', (error) => {
      console.error('[PAGE ERROR]', error.message);
    });

    // Failed network requests
    page.on('requestfailed', (request) => {
      console.error(
        '[REQUEST FAILED]',
        request.url(),
        request.failure()?.errorText || 'Unknown error'
      );
    });

    console.log('🌐 Opening http://localhost:3000 ...');

    const response = await page.goto('http://localhost:3000', {
      waitUntil: 'networkidle',
      timeout: 10000,
    });

    // Check HTTP response
    if (!response) {
      throw new Error('No response received from the server.');
    }

    console.log(`✅ Server responded with status: ${response.status()}`);

    if (!response.ok()) {
      throw new Error(
        `Server returned HTTP ${response.status()} ${response.statusText()}`
      );
    }

    // Give the application a moment to finish rendering
    await page.waitForTimeout(1000);

    console.log(`📄 Page title: ${await page.title()}`);
    console.log(`🔗 Current URL: ${page.url()}`);

    console.log('✅ Test completed successfully.');
  } catch (error) {
    console.error('❌ Test failed:', error.message);

    // Capture a screenshot to help debug failures
    if (browser) {
      try {
        const pages = browser.contexts()[0]?.pages() || [];

        if (pages.length > 0) {
          await pages[0].screenshot({
            path: 'test-server-error.png',
            fullPage: true,
          });

          console.log('📸 Error screenshot saved as test-server-error.png');
        }
      } catch (screenshotError) {
        console.error(
          '⚠️ Could not save screenshot:',
          screenshotError.message
        );
      }
    }

    process.exitCode = 1;
  } finally {
    if (browser) {
      await browser.close();
      console.log('🛑 Browser closed.');
    }
  }
})();
