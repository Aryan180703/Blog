<div class="container mt-4">
  <h2>Products</h2>
  <button class="btn btn-primary mb-3" (click)="loadProducts()">Refresh</button>
  <div *ngIf="!products().length" class="text-center">
    <div class="spinner-border" role="status"></div>
  </div>
  <div class="row">
    <div class="col-md-4" *ngFor="let product of products()">
      <div class="card mb-3 shadow-sm">
        <div class="card-body">
          <h5 class="card-title">{{ product.title }}</h5>
          <p class="card-text">ID: {{ product.id }}</p>
        </div>
      </div>
    </div>
  </div>
</div>






File: src/app/product-list/product-list.component.css (optional polish)Code:.card:hover {
  transform: scale(1.02);
  transition: transform 0.2s;
}



2.4 Update App Component (10 Min)File: src/app/app.component.tsCode:import { Component } from '@angular/core';
import { ProductListComponent } from './product-list/product-list.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [ProductListComponent],
  template: '<app-product-list></app-product-list>'
})
export class AppComponent {}



File: src/app/app.config.tsCode:import { ApplicationConfig } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';
import { provideRouter } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [provideRouter(routes), provideHttpClient()]
};Explanation (with JS/TS):




Step 3: Polish and Test (20 Min)What? Customize for your /api/Products and ensure a pro demo.Tasks:Update API:In product.service.ts:private apiUrl: string = 'YOUR_API_URL';Add auth if needed (see service code).Customize Fields:In product-list.component.html:<h5 class="card-title">{{ product.name }}</h5>
<p class="card-text">Price: ${{ product.price }}</p>JS Explanation: product.name accesses an object property, like:let product = { name: 'Laptop' };
console.log(product.name);Add image if available:<img *ngIf="product.image" [src]="product.image" class="card-img-top" alt="Product">JS Explanation: [src] sets the image URL, like:document.getElementById('img').src = product.image;Improve Types (optional, for safety):In product.service.ts:export interface Product {
  id: number;
  name: string;
  price: number;
  image?: string; // Optional
}

getProducts(): Observable<Product[]> {
  return this.http.get<Product[]>(this.apiUrl);
}TS Explanation: interface defines the product shape, like a JS object with typed properties. ? means optional. In JS, no types:let product = { id: 1, name: 'Laptop', price: 999 };In product-list.component.ts:import { Product } from '../services/product.service';
products = signal<Product[]>([]);TS Explanation: Uses the Product type for safety.Test:Run ng serve.Check products load. If errors (e.g., CORS, 401), note them (F12 console).JS Analogy: Like debugging fetch:fetch(url).catch(err => console.error(err));Troubleshooting:CORS: Backend may block requests. Use JSONPlaceholder or ask for CORS fix.401: Verify JWT token.Wrong Fields: Check API response (e.g., name vs. title) in console.For EShoppingZone: A polished product list (e.g., cards with names, prices, images) wows your HR/mentor.





