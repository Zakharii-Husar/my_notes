# Refactor Strategy

## 0. Update Stack? Or maybe no!

## 1. Get a full copy with Duplicator (production → your laptop)

## 2. Run it locally (Ubuntu) without Docker

## 3. Build + activate a child theme locally (inherits parent automatically)

## 4. "Testing immediately locally" (fast feedback loop)

## 5. Deploy the child theme to production (zip upload)

So updates are:

- Switch production back to the parent theme temporarily
- Delete the old child theme (Theme details → Delete)
- Upload new zip
- Activate child theme again

(Keep old zip files for rollback.)

## 6. Rollback plan (your step 5)

Rollback is instant:

**wp-admin** → **Appearance** → **Themes** → activate parent theme

No DB restore needed.
