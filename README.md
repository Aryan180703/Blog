<div class="container">
  <div *ngIf="isLoading" class="text-center mt-3">
    <h5>Loading...</h5>
  </div>
  <div *ngIf="!isLoading" class="row">
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
  <div *ngIf="!isLoading" class="d-flex justify-content-center mt-3">
    <button class="btn btn-primary me-2" (click)="prevPage()" [disabled]="currentPage === 1">Previous</button>
    <span>Page {{ currentPage }} of {{ totalPages }}</span>
    <button class="btn btn-primary ms-2" (click)="nextPage()" [disabled]="currentPage === totalPages">Next</button>
  </div>
</div>