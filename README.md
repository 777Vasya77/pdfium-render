# Idiomatic Rust bindings for Pdfium

`pdfium-render` provides an idiomatic high-level Rust interface around the low-level bindings to
Pdfium exposed by the excellent `pdfium-sys` crate.

```
    // Exports each page in the given test PDF file as a separate JPEG image.

    use pdfium_render::{pdfium::Pdfium, bitmap::PdfBitmapRotation, bitmap_config::PdfBitmapConfig};
    use image::ImageFormat;

    // Bind to the system-provided Pdfium library.
    
    let pdfium = Pdfium::new(Pdfium::bind_to_system_library().unwrap());

    // Load a PDF file.
    
    let document = pdfium.load_pdf_from_file("test.pdf", None).unwrap();
    
    // Set our desired bitmap rendering options.
 
    let bitmap_render_config = PdfBitmapConfig::new()
        .set_target_width(2000)
        .set_maximum_height(2000)
        .rotate_if_landscape(PdfBitmapRotation::Degrees90, true);

    // Render each page to a bitmap, saving each as a JPEG.
 
    document.pages().iter().for_each(|page| {
        page.get_bitmap_with_config(&bitmap_render_config).unwrap()
            .as_image() // Renders this page to an Image::DynamicImage
            .as_bgra8().unwrap()
            .save_with_format(format!("test-page-{}.jpg", page.index()), ImageFormat::Jpeg).unwrap();
    });
```

More examples, demonstrating page rendering, text extraction, page object introspection, and
compiling to WASM are available at <https://github.com/ajrcarey/pdfium-render/tree/master/examples>.

In addition to providing a more natural interface to Pdfium, `pdfium-render` differs from
`pdfium-sys` in several other important ways:

* `pdfium-render` uses `libloading` to late bind to a Pdfium library at run-time, whereas
  `pdfium-sys` binds to a library at compile-time. By binding to Pdfium at run-time instead
  of compile-time, `pdfium-render` can dynamically switch between bundled libraries and
  system libraries and provide idiomatic Rust error handling in situations where a Pdfium
  library is not available.
* Late binding to Pdfium means that `pdfium-render` can be compiled to WASM for running in a
  browser; this is not possible with `pdfium-sys`.
* Pdfium is composed as a set of separate modules, each covering a different aspect of PDF creation,
  rendering, and editing. `pdfium-sys` only provides bindings for the subset of functions exposed
  by Pdfium's view module; `pdfium-render` aims to ultimately provide bindings to all non-interactive
  functions exposed by all Pdfium modules, including document creation and editing functions.
  This is a work in progress. 
* Pages rendered by Pdfium can be exported as instances of `Image::DynamicImage` for easy,
  idiomatic post-processing.

## What's new

Version 0.5.3 adds bindings for `FPDFBookmark_*()`, `FPDFPageObj_*()`, `FPDFText_*()`, and `FPDFFont_*()` functions and adds the `PdfPageObjects`, `PdfPageText`, and `PdfBookmarks` collections
to the high-level idiomatic interface. It is now possible to extract text from PDF pages and page objects.
 
## Porting existing Pdfium code from other languages

The high-level idiomatic Rust interface provided by the `Pdfium` struct is entirely optional;
the `Pdfium` struct wraps around raw FFI bindings defined in the `PdfiumLibraryBindings`
trait, and it is completely feasible to simply use the FFI bindings directly
instead of the high level interface. This makes porting existing code that calls FPDF_* functions
trivial, while still gaining the benefits of late binding and WASM compatibility.
For instance, the following code snippet (taken from a C++ sample):

```
    string test_doc = "test.pdf";

    FPDF_InitLibrary();
    FPDF_DOCUMENT doc = FPDF_LoadDocument(test_doc, NULL);
    // ... do something with doc
    FPDF_CloseDocument(doc);
    FPDF_DestroyLibrary();
```

would translate to the following Rust code:

```
    let bindings = Pdfium::bind_to_system_library().unwrap();
    
    let test_doc = "test.pdf";

    bindings.FPDF_InitLibrary();
    let doc = bindings.FPDF_LoadDocument(test_doc, None);
    // ... do something with doc
    bindings.FPDF_CloseDocument(doc);
    bindings.FPDF_DestroyLibrary();
```

## External Pdfium builds

`pdfium-render` does not include Pdfium itself. You can either bind to a system-provided library
or package an external build of Pdfium alongside your Rust application. When compiling to WASM,
packaging an external build of Pdfium as a separate WASM module is essential.

* Native builds of Pdfium for all major platforms _except_ WASM: <https://github.com/bblanchon/pdfium-binaries/releases>
* WASM builds of Pdfium, suitable for deploying alongside `pdfium-render`: <https://github.com/paulo-coutinho/pdfium-lib/releases>

## Compiling to WASM

See <https://github.com/ajrcarey/pdfium-render/tree/master/examples> for a full example that shows
how to bundle a Rust application using `pdfium-render` alongside a pre-built Pdfium WASM module for
inspection and rendering of PDF files in a web browser.

## Development status

The initial focus of this crate has been on rendering pages in a PDF file; consequently, FPDF_*
functions related to bitmaps and rendering have been prioritised. By 1.0, the functionality of all
FPDF_* functions exported by all Pdfium modules will be available, with the exception of certain
functions specific to interactive scripting, user interaction, and printing.

If you need a function that is not currently exposed, just raise an issue.

## Version history

* 0.5.3: adds bindings for `FPDFBookmark_*()`, `FPDFPageObj_*()`, `FPDFText_*()`, and `FPDFFont_*()` functions, exposes `PdfPageObjects`, `PdfPageText`, and `PdfBookmarks` collections
* 0.5.2: adds bindings for `FPDF_GetPageBoundingBox()`, `FPDFDoc_GetPageMode()`, `FPDFPage_Get*Box()`, and `FPDFPage_Set*Box()` functions, exposes `PdfPageBoundaries` collection
* 0.5.1: adds bindings for `FPDFPage_GetRotation()` and `FPDFPage_SetRotation()` functions, exposes `PdfMetadata` collection
* 0.5.0: adds rendering of annotations and form field elements, thanks to an excellent contribution from <https://github.com/inzanez>
* 0.4.2: bug fixes in `PdfBitmapConfig` implementation
* 0.4.1: improvements to documentation and READMEs
* 0.4.0: initial release