import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

// Define SMD response structure
export interface ApiResponse {
  success: boolean;
  message: string;
  data: Product[];
}

// Define Product structure
export interface Product {
  id: number;
  name: string;
  price: number;
  // Add other fields (e.g., image) if your API includes them
}

@Injectable({ providedIn: 'root' })
export class ProductService {
  private apiUrl: string = 'http://localhost:5000/api/Products'; // Your backend URL

  constructor(private http: HttpClient) {}

  getProducts(): Observable<ApiResponse> {
    console.log('Calling API:', this.apiUrl);
    return this.http.get<ApiResponse>(this.apiUrl);
  }
}





Step 2: Update ProductListComponentHandle the SMD response and add the search bar.File: src/app/product-list/product-list.component.tsimport { Component, signal } from '@angular/core';
import { ProductService, Product } from '../services/product.service';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { FilterPipe } from '../pipes/filter.pipe';

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [CommonModule, FormsModule, FilterPipe],
  templateUrl: './product-list.component.html',
  styleUrls: ['./product-list.component.css']
})
export class ProductListComponent {
  products = signal<Product[]>([]);
  searchTerm = '';

  constructor(private productService: ProductService) {
    this.loadProducts();
  }

  loadProducts() {
    console.log('Fetching products...');
    this.productService.getProducts().subscribe({
      next: (response) => {
        console.log('Response:', response);
        if (response.success) {
          this.products.set(response.data.slice(0, 5));
          console.log('Products set:', this.products());
        } else {
          console.error('API failed:', response.message);
          this.products.set([]);
        }
      },
      error: (err) => {
        console.error('API Error:', err);
        this.products.set([]);
      }
    });
  }
}



<div class="container mt-4">
  <h2>EShoppingZone Products</h2>
  <input [(ngModel)]="searchTerm" class="form-control mb-3" placeholder="Search products">
  <button class="btn btn-primary mb-3" (click)="loadProducts()">Refresh</button>
  <div *ngIf="!products().length" class="text-center">
    <div class="spinner-border" role="status"></div>
  </div>
  <div class="row">
    <div class="col-md-4" *ngFor="let product of products() | filter:searchTerm">
      <div class="card mb-3 shadow-sm">
        <img *ngIf="product.image" [src]="product.image" class="card-img-top" alt="Product">
        <div class="card-body">
          <h5 class="card-title">{{ product.name }}</h5>
          <p class="card-text">Price: ${{ product.price }}</p>
          <p class="card-text">ID: {{ product.id }}</p>
        </div>
      </div>
    </div>
  </div>
</div>




Step 4: Add Filter PipeFile: src/app/pipes/filter.pipe.tsng generate pipe pipes/filterimport { Pipe, PipeTransform } from '@angular/core';
import { Product } from '../services/product.service';

@Pipe({ name: 'filter', standalone: true })
export class FilterPipe implements PipeTransform {
  transform(items: Product[], searchTerm: string): Product[] {
    if (!items || !searchTerm) return items;
    return items.filter(item => 
      item.name.toLowerCase().includes(searchTerm.toLowerCase())
    );
  }
}




var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers();

builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAngular", builder =>
    {
        builder.WithOrigins("http://localhost:4200")
               .AllowAnyMethod()
               .AllowAnyHeader();
    });
});

var app = builder.Build();

// Configure middleware
app.UseHttpsRedirection();
app.UseCors("AllowAngular");
app.UseAuthorization();
app.MapControllers();

app.Run();






getProducts(): Observable<ApiResponse> {
  console.log('Calling API:', this.apiUrl);
  return this.http.get<ApiResponse>(this.apiUrl, {
    headers: { Authorization: 'Bearer YOUR_TOKEN' }
  });
}