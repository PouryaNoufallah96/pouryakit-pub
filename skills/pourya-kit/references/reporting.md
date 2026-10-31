# Reporting

`PouryaKit.Reporting`: one renderer, `IReportRenderer`, that renders a Module's `ReportLayout` to a PDF, a CSV file or a spreadsheet in memory and returns a `Result`. The layout stays in the Module; the Kit supplies the primitives, the font, the branding and the three formats. Optional, namespace `PouryaKit.Reporting`; it references `PouryaKit.Domain` only, and stores nothing: the caller stores the rendered bytes, through [storage.md](storage.md).

- [Registering](#registering)
- [A Module's layout](#a-modules-layout)
- [Rendering and storing](#rendering-and-storing)
- [The three formats](#the-three-formats)
- [Limits](#limits)
- [Testing](#testing)

## Registering

In the Composition Root of each Module that renders reports:

```csharp
using Microsoft.Extensions.DependencyInjection.Extensions;
using PouryaKit.Reporting;

module.TryAddSingleton<IReportRenderer, ReportRenderer>();
```

- `ReportRenderer` keeps no state of its own, so one instance serves every caller at once; `TryAddSingleton` lets several Modules register it. The package has no registration call and no options.
- The PDF writer sets PDFsharp's process-wide font resolver to the Roboto faces the package embeds, the first time a PDF renders, unless the Host set its own resolver before; that one then resolves the report's fonts.

## A Module's layout

A layout is a value the Module builds from what it read: a title, an introduction of headings and paragraphs, one table of text cells, and the branding. The Module formats every value, such as a number, an amount or a date, as the text the report shows, in the culture it chooses:

```csharp
using System.Globalization;
using PouryaKit.Reporting;

internal static class SalesReport
{
    private static readonly ReportBranding Branding = new()
    {
        Logo = File.ReadAllBytes(Path.Combine(AppContext.BaseDirectory, "Reports", "logo.png")),
        Accent = new(0x1F, 0x4E, 0x79),
    };

    public static ReportLayout For(DateOnly month, IReadOnlyList<SalesLine> lines) =>
        new(
            "Monthly sales",
            new ReportTable(
                [new("Product"), new("Quantity", ReportAlignment.Right), new("Amount", ReportAlignment.Right)],
                [
                    .. lines.Select(line => (IReadOnlyList<string>)
                    [
                        line.Product,
                        line.Quantity.ToString(CultureInfo.InvariantCulture),
                        line.Amount.ToString("N2", CultureInfo.InvariantCulture),
                    ]),
                ]
            )
        )
        {
            Introduction =
            [
                new ReportHeading(month.ToString("MMMM yyyy", CultureInfo.InvariantCulture)),
                new ReportParagraph("Every order settled this month."),
            ],
            PageHeader = "Acme Shop",
            PageFooter = "Confidential",
            Branding = Branding,
        };
}

internal sealed record SalesLine(string Product, int Quantity, decimal Amount);
```

- `ReportLayout(Title, Table)`: `Introduction` (`ReportHeading` and `ReportParagraph`, in order), `PageHeader`, `PageFooter` and `Branding` are optional.
- `ReportTable(Columns, Rows)`: each `ReportColumn` has a header and an alignment, `Left` by default; each row has exactly one cell per column.
- `ReportBranding`: `Logo`, a PNG or JPEG image, and `Accent`, a `ReportColor`, dark grey by default, for the title and the table's header row.

## Rendering and storing

`Render(layout, format)` returns a `RenderedReport` with the file's `Content`, its `ContentType` and its `FileExtension`, ready for `IFileStorage.UploadAsync`. Store first, then record the key, as [storage.md](storage.md#store-then-record) says:

```csharp
var rendered = renderer.Render(SalesReport.For(command.Month, lines), ReportFormat.Pdf);
if (rendered.IsFailure)
{
    return rendered;
}

var report = rendered.Value;
var key = $"reports/sales/{Guid.CreateVersion7()}{report.FileExtension}";
var stored = await files.UploadAsync(key, new MemoryStream(report.Content), report.ContentType, cancellationToken);
```

`Render` refuses a layout it cannot draw, renders nothing, and returns an `ErrorKind.Validation` Error:

- `ReportErrors.TableShape` (`Reporting.TableShape`, target `Table`): the table has no column, or a row whose number of cells differs from its number of columns.
- `ReportErrors.LogoFormat` (`Reporting.LogoFormat`, target `Logo`): the logo is not a PNG or JPEG image PDFsharp can load, its signature checked, then its header and dimensions read, so a file cut short is refused, where MigraDoc would have drawn a placeholder in the PDF. A JPEG cut inside its image data still loads, since a PDF carries a JPEG as it is.

Rendering is synchronous and in memory, so the whole file is one byte array; render a large report from a background command, as the report Module does ([reporting-module.md](reporting-module.md)).

## The three formats

| `ReportFormat` | Content type and extension | Holds |
| - | - | - |
| `Pdf` | `application/pdf`, `.pdf` | An A4 portrait document: the logo, the title, the introduction and the table, whose header row repeats on every page, with the page header, and the page footer before `page / pages`, on every page. MigraDoc and PDFsharp, the Roboto font embedded, so the file renders where no font is installed, the Linux CI runner included. |
| `Csv` | `text/csv`, `.csv` | The table alone, headers first: RFC 4180 through CsvHelper with the invariant culture, in UTF-8 with a byte order mark, so a spreadsheet application reads non-ASCII text as written. A cell that starts with `=`, `+`, `-`, `@`, a tab or a carriage return is written with a leading apostrophe (`'-4+cmd`), so a spreadsheet application never runs it as a formula, unless the whole cell is a number in the invariant culture (`decimal.TryParse` with `NumberStyles.Number`), such as `-4` or `+1,250.50`, which stays as written and computable. An empty field is quoted (`""`), so the empty row of a one-column table survives a reader that skips blank lines. |
| `Spreadsheet` | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`, `.xlsx` | The table alone on one worksheet named `Report`, headers first, every cell an inline string exactly as the Module formatted it: text, numbers included, never a formula. A literal `_xHHHH_` is written `_x005F_xHHHH_` and a character XML cannot hold, such as a control character, as `_xHHHH_`, so a spreadsheet application reads each cell back verbatim. The Open XML SDK. |

## Limits

- One table per report, of text cells; no charts, images in cells, merged cells or cell styles.
- One font, Roboto 2.138 regular and bold, with italic simulated: it holds Latin, Greek and Cyrillic glyphs, not Persian or Arabic, and the PDF writer lays out no right-to-left text.
- PDFsharp is pinned at 6.2.4, its last MIT release: 6.5.0 and later are AGPL-3.0-only, and the Kit takes no reciprocally licensed package.
- QuestPDF and ClosedXML are not used (licence), and no second file library.

## Testing

The Kit's tests render one layout to each format and read it back with that format's standard reader: the PDF with PDFsharp's `PdfReader` (its header, page count, logo and each page's fonts embedded with their font program), the spreadsheet with the Open XML SDK's `SpreadsheetDocument.Open` and its `OpenXmlValidator` (the cells as rows, escape-like text and control characters read back verbatim), and the CSV with CsvHelper's `CsvReader` (the rows, the escaped formulas, the numbers kept, the byte order mark and a one-column table's empty rows). A Module's own report test renders its layout the same way: build the layout from fixed rows, render it, and read back the cells it promises.
