# SOFTWARE BUG FIX & CODE REVIEW REPORT

**Projects:** Handwriting Recognition (OCR) Application and Book Management System  
**Review Type:** Bug Identification, Root Cause Analysis, Code-Level Fix Review  
**Technology Stack:** Python, FastAPI, OpenCV, Tesseract OCR, JavaScript, Node.js, Express.js, SQLite  
**Status:** Code review completed; final verification required

---

## 1. Executive Summary

This report documents 12 identified defects across two software applications. The defects affected OCR text recognition, image preprocessing, file downloads, book filtering, API routing, book updates, stock adjustments, dashboard statistics, frontend search, and server startup logging.

The review focused on identifying the underlying causes rather than simply modifying test expectations. It also examined whether the uploaded source code actually contains the reported corrections.

The review identified two distinct project versions:

- **`_node_aapp-fixed.zip`:** Contains several corrected Book Management System operations, but the OCR-related defects shown in the report remain in its source code.
- **`aapp_fixed(2).zip`:** Contains important OCR corrections, but several Book Management System defects have reappeared in its source code.

Therefore, the final deliverable should combine the correct changes from both versions, remove the regressions, and pass the relevant automated and real-sample tests.

## 2. Project Overview

### 2.1 Handwriting Recognition Application

The application accepts an uploaded image, preprocesses it, extracts text using Tesseract OCR, and returns the recognized text for viewing or downloading.

**Main technologies:**
- Python for application logic
- FastAPI for HTTP endpoints
- OpenCV and NumPy for image processing
- Tesseract OCR through `pytesseract`
- HTML, CSS, and JavaScript for the frontend

### 2.2 Book Management System

The application manages a book catalogue, including book details, categories, search, stock quantities, and dashboard statistics.

**Main technologies:**
- Node.js and Express.js for the backend
- JavaScript for controllers, services, and repositories
- SQLite for persistent data storage
- HTML, CSS, and JavaScript for the frontend
- Jest and Supertest for automated testing

---

## 3. Detailed Bug-Fix Report: OCR Application

### Bug 1: Empty text download

**Affected file:** `python/app/utils/files.py`

**Problem:** The result-reading function opens a text file using write mode (`w+`). This mode truncates the existing file before reading it, potentially producing an empty download.

**Root cause:** The same file-opening mode was used for a read operation, even though the application needed to preserve and retrieve the previously saved OCR output.

**Required correction:** Use read mode (`r`) with UTF-8 encoding.

```python
with path.open("r", encoding="utf-8") as handle:
    return handle.read()
```

**Expected result:** The downloaded text file should contain the recognized OCR output instead of being empty.

**Verification:** Recognize text from an image, download the result, and confirm that the file contains the same text displayed by the application.

**Archive status:** This defect remains in `_node_aapp-fixed.zip`; it is corrected in neither version based on the inspected function.

### Bug 2: Image distortion during resizing

**Affected file:** `python/app/preprocessing/image_ops.py`

**Problem:** Images with different dimensions were forced into a fixed 800 × 600 resolution.

**Root cause:** Resizing to fixed dimensions changes the image's aspect ratio whenever the original image has a different width-to-height ratio. This can distort handwritten characters and reduce OCR readability.

**Required correction:** Calculate a scaling factor that respects the maximum width and height while preserving the original aspect ratio.

```python
scale = min(max_width / width, max_height / height)
new_size = (
    max(1, round(width * scale)),
    max(1, round(height * scale))
)
```

The resizing implementation should also select an appropriate interpolation method for shrinking or enlarging images.

**Expected result:** Images retain their original proportions while remaining within the configured size limits.

**Verification:** Test wide, tall, and standard images. Confirm that none is stretched and that all remain within the maximum dimensions.

**Archive status:** Implemented in `aapp_fixed(2).zip`; not implemented in `_node_aapp-fixed.zip`.

### Bug 3: Incorrect thresholding for OCR

**Affected file:** `python/app/preprocessing/image_ops.py`

**Problem:** The preprocessing pipeline used inverted binary thresholding, producing white text on a black background.

**Root cause:** The thresholding configuration did not match the intended OCR input format of black text on a white background.

**Required correction:** Use standard binary thresholding with Otsu's automatic threshold selection:

```python
_, binary = cv2.threshold(
    blur,
    0,
    255,
    cv2.THRESH_BINARY + cv2.THRESH_OTSU
)
```

**Expected result:** The preprocessed image should have the intended black-text-on-white-background appearance for the chosen OCR workflow.

**Verification:** Inspect the processed image visually and test OCR on samples with dark text, light backgrounds, and varying contrast.

**Archive status:** The correction is present in `aapp_fixed(2).zip` according to the code comparison and absent from `_node_aapp-fixed.zip`.

### Bug 4: OCR configured for single-line text

**Affected file:** `python/app/recognition/ocr.py`

**Problem:** Tesseract was configured with `--psm 7`, which assumes a single line of text. This is unsuitable for handwritten notes containing multiple lines.

**Root cause:** The OCR page-segmentation mode did not match the structure of the uploaded documents.

**Required correction:**

```python
return "--oem 3 --psm 6"
```

Page Segmentation Mode 6 treats the image as a single uniform block of text. It is a more appropriate starting point for multi-line notes, although performance depends on the image layout.

**Expected result:** Tesseract can process multiple lines instead of being restricted to a single-line layout assumption.

**Verification:** Test one-line text, multi-line notes, and images containing paragraphs. Compare the recognized output for each sample.

**Archive status:** Implemented in `aapp_fixed(2).zip`; not implemented in `_node_aapp-fixed.zip`.

### Bug 5: Incorrect argument order in the OCR pipeline

**Affected file:** `python/app/services/pipeline.py`

**Problem:** The pipeline passed the preprocessed image and resized image to `prepare_for_ocr()` in the wrong order.

**Root cause:** The function expects the original or resized image as its first argument and the preprocessed image as its second argument. The incorrect call could cause the pipeline to use the unprocessed image for recognition.

**Required correction:**

```python
ocr_input = prepare_for_ocr(resized, preprocessed)
```

**Expected result:** The OCR engine receives the intended preprocessed image, enabling the thresholding and noise-reduction steps to contribute to recognition.

**Verification:** Check the image passed into `extract_text()` and compare OCR results with and without preprocessing.

**Archive status:** Corrected in `aapp_fixed(2).zip`; incorrect in `_node_aapp-fixed.zip`.

---

## 4. Detailed Bug-Fix Report: Book Management System

### Bug 6: Incorrect category filtering

**Affected file:** `nodejs/src/repositories/bookRepository.js`

**Problem:** Selecting a category could return books from every other category instead of books belonging to the selected category.

**Root cause:** The SQL filter used the not-equal operator (`!=`) instead of the equality operator (`=`).

**Required correction:**

```javascript
conditions.push("category = ?");
params.push(category);
```

**Expected result:** Selecting Science returns Science books only, and selecting another category returns books belonging to that category.

**Verification:** Test multiple categories and confirm that every returned book matches the selected category.

**Archive status:** Correct in `_node_aapp-fixed.zip`; incorrect in `aapp_fixed(2).zip`.

### Bug 7: Search route conflicts with the book ID route

**Affected file:** `nodejs/src/routes/books.js`

**Problem:** The `/search` endpoint was declared after the dynamic `/:id` route.

**Root cause:** Express evaluates matching routes in declaration order. The dynamic route could interpret `search` as a book ID and prevent the intended search handler from running.

**Required correction:**

```javascript
router.get("/search", controller.search);
router.get("/:id", controller.getById);
```

**Expected result:** Requests to `/api/books/search` reach the search controller, while `/api/books/123` reaches the book-by-ID controller.

**Verification:** Test both endpoints and confirm that each returns the correct response.

**Archive status:** Correct in `_node_aapp-fixed.zip`; incorrect in `aapp_fixed(2).zip`.

### Bug 8: Book price and publication year were swapped

**Affected file:** `nodejs/src/controllers/bookController.js`

**Problem:** Updating a book could store its publication year in the price field and its price in the publication-year field.

**Root cause:** The controller passed arguments to the service in an order that did not match the service function's expected parameters.

**Required correction:** Keep the order consistent throughout the controller and service layers:

1. Title
2. Author
3. ISBN
4. Category
5. Publication year
6. Price
7. Quantity

**Expected result:** Editing a book preserves the correct data types and values for publication year and price.

**Verification:** Update a book using a publication year such as `2001` and a price such as `12.5`. Confirm that the returned record preserves both values correctly.

**Archive status:** Correct in `_node_aapp-fixed.zip`; incorrect in `aapp_fixed(2).zip`.

### Bug 9: Stock decrement operation was incorrect

**Affected file:** `nodejs/src/services/stockService.js`

**Problem:** The stock-decrement operation could increase inventory rather than subtracting the requested quantity. The negative-stock validation was also affected by the incorrect calculation.

**Root cause:** The adjustment sign was handled incorrectly, making the resulting stock calculation inconsistent with the requested operation.

**Required correction:**

```javascript
const next = book.quantity + delta;

if (next < 0) {
    const error = new Error("Insufficient stock");
    error.status = 400;
    throw error;
}
```

Here, `delta` must be a signed adjustment: a positive value increases stock and a negative value decreases it.

**Expected result:** Stock changes correctly in both directions, and the system rejects an adjustment that would produce a negative quantity.

**Verification:** Test an increase of one, a decrease of one, and a decrement that would take stock below zero.

**Archive status:** Correct in `_node_aapp-fixed.zip`; incorrect in `aapp_fixed(2).zip`.

### Bug 10: Dashboard stock totals were incorrect

**Affected file:** `nodejs/src/repositories/statsRepository.js`

**Problem:** The dashboard's available-copy count ignored books with exactly one copy in stock.

**Root cause:** The SQL aggregation counted quantities only when `quantity > 1`, excluding valid inventory with a quantity of one.

**Required correction:**

```sql
COALESCE(SUM(quantity), 0) AS available_copies
```

**Expected result:** The dashboard's available-copy count equals the sum of the stock quantities of all books, including books with zero or one copy.

**Verification:** Compare the dashboard total with the sum of the `quantity` values returned by the book-list endpoint.

**Archive status:** Correct in `_node_aapp-fixed.zip`; incorrect in `aapp_fixed(2).zip`.

### Bug 11: Frontend search used outdated input

**Affected file:** `nodejs/frontend/js/app.js`

**Problem:** Search results could lag behind the characters entered by the user.

**Root cause:** The frontend used the previous search value rather than the current input value when making a request.

**Required correction:** Read the current search input when generating the request and use that value to construct the query parameters.

```javascript
async function loadBooks(query = searchInput.value.trim()) {
    const params = new URLSearchParams();

    if (query) {
        params.set("q", query);
    }

    // Continue with the API request.
}
```

**Expected result:** Search results correspond to the current query entered by the user.

**Verification:** Enter a word one character at a time, clear the search box, and test searches matching titles, authors, and ISBN values.

**Archive status:** The direct-current-value approach is present in `_node_aapp-fixed.zip`. The previous-value logic remains in `aapp_fixed(2).zip`.

### Bug 12: Server startup did not print the application URL

**Affected file:** `nodejs/src/server.js`

**Problem:** The server started without displaying a helpful URL in the terminal.

**Root cause:** The `listen()` callback did not include a startup message.

**Required correction:**

```javascript
app.listen(config.port, () => {
    console.log(`Server running at http://localhost:${config.port}`);
});
```

**Expected result:** Developers can immediately see the local URL after starting the application.

**Verification:** Run `npm start` and confirm that the terminal prints the correct URL and port.

**Archive status:** Correct in `_node_aapp-fixed.zip`; missing in `aapp_fixed(2).zip`.

---

## 5. Cross-Version Verification Summary

| Defect | `_node_aapp-fixed.zip` | `aapp_fixed(2).zip` |
|---|---|---|
| 1. Empty OCR download | Not fixed | Not fixed |
| 2. Image aspect ratio | Not fixed | Fixed |
| 3. Threshold inversion | Not fixed | Fixed |
| 4. Multi-line OCR mode | Not fixed | Fixed |
| 5. OCR argument order | Not fixed | Fixed |
| 6. Category filtering | Fixed | Regression present |
| 7. Search route order | Fixed | Regression present |
| 8. Update argument order | Fixed | Regression present |
| 9. Stock adjustment | Fixed | Regression present |
| 10. Inventory statistics | Fixed | Regression present |
| 11. Search input handling | Fixed | Regression present |
| 12. Startup URL logging | Fixed | Missing |

*This table reflects source-code inspection, not successful execution of all test cases.*

## 6. Testing and Quality Assurance Plan

Before declaring the project fully fixed, the following checks should be completed.

### OCR application

- [ ] Confirm that text downloads contain the recognized output.
- [ ] Verify aspect-ratio preservation on wide, tall, and standard images.
- [ ] Inspect the output of image preprocessing.
- [ ] Test single-line and multi-line handwriting samples.
- [ ] Verify that the OCR pipeline passes the intended image to Tesseract.
- [ ] Test invalid images, unsupported file formats, and oversized uploads.
- [ ] Compare OCR output against manually transcribed sample text.

### Book Management System

- [ ] Test category filtering for every available category.
- [ ] Verify the `/api/books/search` route.
- [ ] Edit a book and confirm that its price and publication year remain correct.
- [ ] Test stock increments, decrements, and negative-stock rejection.
- [ ] Compare dashboard inventory totals with actual book quantities.
- [ ] Test current-value search behavior.
- [ ] Verify that the server prints the correct startup URL.
- [ ] Run the automated test suite and investigate every failure.

**Recommended OCR evaluation metrics:** Character Error Rate (CER) and Word Error Rate (WER). These measure the difference between recognized text and manually verified reference text. They provide a more meaningful assessment of OCR quality than tests that merely check whether a response exists.

## 7. Recommended Finalization Procedure

1. Use `aapp_fixed(2).zip` as the starting point for its corrected OCR preprocessing and pipeline changes.
2. Restore the six correct Book Management System changes from `_node_aapp-fixed.zip`.
3. Fix the empty-download issue in `python/app/utils/files.py`.
4. Add or strengthen regression tests for all 12 defects.
5. Run the Python and Node.js test suites.
6. Test the OCR application with the supplied sample images and verify downloaded text files.
7. Run the Book Management System manually and validate filtering, search, editing, stock operations, and dashboard totals.
8. Package the consolidated, tested version and update the README with the final setup instructions.

## 8. Conclusion

The defects reviewed in these applications involve both functional correctness and reliability. The OCR fixes address image quality, text extraction configuration, pipeline integration, and file handling. The Book Management System fixes address query logic, route matching, parameter consistency, inventory calculations, frontend behavior, and developer usability.

The source review confirms that many individual corrections are present across the two uploaded versions. However, changes made in one version are missing or regressed in the other. The project should therefore be considered **partially corrected, pending consolidation and test verification**, rather than fully validated.

The final acceptance criterion is straightforward: all 12 defects must be corrected in one consistent codebase, the relevant regression tests must pass, and real-image OCR results and book-management operations must behave as expected.
