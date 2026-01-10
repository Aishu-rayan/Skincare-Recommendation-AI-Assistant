# Database Cleanup Report

Generated: 2026-01-10 23:10

## Summary

- Total products now: **2,522**
- Total reviews now: **238,929**
- Orphan reviews (reviews whose `product_id` is not present in `product_info`): **0**

## Products by primary_category (post-cleanup)

- **Skincare**: 2,518
- **Gifts**: 4

## Notes on moves/deletions performed

- Moved **Men → Skincare**: 32 products (mapped into existing Skincare categories: Eye Care, Cleansers, Moisturizers, Sunscreen).
- Moved **Mini Size/Skincare → Skincare**: 66 products (mapped via product name keywords into existing Skincare categories).
- Deleted categories: Bath & Body, Fragrance, Hair, Makeup, Men (remaining), Mini Size (remaining), Tools & Brushes.
- Deleted associated reviews for deleted products: **0** (no reviews referenced the deleted product_ids).

## Skincare category breakdown (post-cleanup)

- Moisturizers → Moisturizers: 427 products
- Treatments → Face Serums: 385 products
- Cleansers → Face Wash & Cleansers: 235 products
- Value & Gift Sets → (null): 196 products
- Eye Care → Eye Creams & Treatments: 179 products
- Masks → Face Masks: 134 products
- Mini Size → (null): 112 products
- Sunscreen → Face Sunscreen: 103 products
- Cleansers → Toners: 83 products
- Moisturizers → Face Oils: 68 products
- Moisturizers → Mists & Essences: 67 products
- Lip Balms & Treatments → (null): 61 products
- Treatments → Facial Peels: 53 products
- High Tech Tools → Anti-Aging: 49 products
- Cleansers → Exfoliators: 47 products
- Wellness → Beauty Supplements: 44 products
- Treatments → Blemish & Acne Treatments: 38 products
- Self Tanners → For Body: 34 products
- Masks → Sheet Masks: 32 products
- Wellness → Facial Rollers: 25 products
- Moisturizers → Night Creams: 20 products
- Self Tanners → For Face: 18 products
- Eye Care → Eye Masks: 14 products
- High Tech Tools → Facial Cleansing Brushes: 13 products
- Cleansers → Makeup Removers: 10 products
- High Tech Tools → Hair Removal: 10 products
- Moisturizers → Decollete & Neck Creams: 10 products
- Sunscreen → Body Sunscreen: 10 products
- Wellness → Holistic Wellness: 10 products
- Cleansers → Face Wipes: 6 products
- High Tech Tools → Teeth Whitening: 5 products
- Cleansers → Blotting Papers: 4 products
- Moisturizers → BB & CC Creams: 4 products
- Cleansers → (null): 3 products
- High Tech Tools → (null): 3 products
- Sunscreen → (null): 3 products
- Moisturizers → (null): 1 products
- Self Tanners → (null): 1 products
- Shop by Concern → Anti-Aging: 1 products

## Review category hierarchy (post-cleanup)

- Skincare → Cleansers → Exfoliators: 7,382 reviews; 15 distinct products
- Skincare → Cleansers → Face Wash & Cleansers: 29,938 reviews; 77 distinct products
- Skincare → Cleansers → Toners: 13,012 reviews; 28 distinct products
- Skincare → Eye Care → Eye Creams & Treatments: 29,650 reviews; 70 distinct products
- Skincare → Lip Balms & Treatments → (null): 7,775 reviews; 16 distinct products
- Skincare → Moisturizers → BB & CC Creams: 893 reviews; 2 distinct products
- Skincare → Moisturizers → Face Oils: 11,832 reviews; 32 distinct products
- Skincare → Moisturizers → Moisturizers: 46,997 reviews; 116 distinct products
- Skincare → Moisturizers → Night Creams: 3,951 reviews; 8 distinct products
- Skincare → Sunscreen → (null): 491 reviews; 1 distinct products
- Skincare → Sunscreen → Face Sunscreen: 16,233 reviews; 44 distinct products
- Skincare → Treatments → Blemish & Acne Treatments: 5,438 reviews; 14 distinct products
- Skincare → Treatments → Face Serums: 65,337 reviews; 161 distinct products
