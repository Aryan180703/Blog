<div class="container">
  <div class="row">
    <div *ngFor="let product of displayedProducts" class="col-md-4 mb-3">
      <div class="card">
        <img [src]="product.imageUrl" class="card-img-top" alt="{{ product.name }}">
        <div class="card-body">
          <h5 class="card-title">{{ product.name }}</h5>
          <p class="card-text">₹{{ product.price }}</p>
          <p class="card-text">{{ product.description }}</p>
        </div>
      </div>
    </div>
  </div>
</div>




"styles": ["node_modules/bootstrap/dist/css/bootstrap.min.css"]



get displayedProducts() {
  const start = (this.currentPage - 1) * this.pageSize;
  return this.products.slice(start, start + this.pageSize);
}





get totalPages() {
  return Math.ceil(this.products.length / this.pageSize);
}

nextPage() {
  if (this.currentPage < this.totalPages) {
    this.currentPage++;
  }
}

prevPage() {
  if (this.currentPage > 1) {
    this.currentPage--;
  }
}







<div class="d-flex justify-content-center mt-3">
  <button class="btn btn-primary me-2" (click)="prevPage()" [disabled]="currentPage === 1">Previous</button>
  <span>Page {{ currentPage }} of {{ totalPages }}</span>
  <button class="btn btn-primary ms-2" (click)="nextPage()" [disabled]="currentPage === totalPages">Next</button>
</div>






import { Component } from '@angular/core';
import { HttpClientModule } from '@angular/common/http';
import { ProductListComponent } from './product-list/product-list.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [HttpClientModule, ProductListComponent],
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {}






<app-product-list></app-product-list>