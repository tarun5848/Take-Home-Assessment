import { chromium, Browser, Page } from 'playwright-chromium';
import * as fs from 'fs';

interface Product {
  name: string;
  price: string;
  link: string;
}

class ECommerceScraper {
  private browser: Browser;
  private page: Page;

  constructor(private baseUrl: string) {}

  async initialize() {
    this.browser = await chromium.launch();
    this.page = await this.browser.newPage();
  }

  async searchAndScrape(searchTerm: string, numberOfProducts: number): Promise<Product[]> {
    const url = `${this.baseUrl}/s?k=${searchTerm}`;
    await this.page.goto(url);

    const products = await this.page.$$('.s-result-item');

    const productData: Product[] = [];

    for (const product of products) {
      const name = await product.$eval('.a-text-normal', el => el.textContent);
      const price = await product.$eval('.a-price .a-offscreen', el => el.textContent);
      const link = await product.$eval('a.a-link-normal', el => el.getAttribute('href'));

      productData.push({ name, price, link });
    }

    productData.sort((a, b) => parseFloat(a.price) - parseFloat(b.price));

    return productData.slice(0, numberOfProducts);
  }

  async close() {
    await this.browser.close();
  }
}

(async () => {
  const scraper = new ECommerceScraper('https://www.amazon.com');
  await scraper.initialize();

  const searchTerm = 'laptop'; // Change this to your desired search term
  const numberOfProducts = 3;

  const products = await scraper.searchAndScrape(searchTerm, numberOfProducts);

  const csvData = products.map(product => `${product.name},${product.price},${searchTerm},${scraper.baseUrl}${product.link}`).join('\n');
  fs.writeFileSync('products.csv', csvData, 'utf-8');

  await scraper.close();
})();
