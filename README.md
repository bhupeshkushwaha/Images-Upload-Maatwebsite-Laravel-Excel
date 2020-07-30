# Images-Upload-Maatwebsite-Laravel-Excel

[Sample Image](https://pin.it/3QHQ93H)

## Import Controller 
```
function import(Request $request) {
        $this->validate($request, [
            'file' => 'required|mimes:xls,xlsx'
        ]);

        $imageSRC = array();
        $spreadsheet = \PhpOffice\PhpSpreadsheet\IOFactory::load($request->file('file'));
        $worksheet = $spreadsheet->getActiveSheet();
        $worksheetArray = $worksheet->toArray();
        array_shift($worksheetArray);

        $worksheetArray = array_map('array_filter', $worksheetArray);
        $worksheetArray = array_filter($worksheetArray);

        foreach ($worksheetArray as $key => $value)
        {
            $worksheet = $spreadsheet->getActiveSheet();
            if (isset($worksheet->getDrawingCollection()[$key])) {
                $drawing = $worksheet->getDrawingCollection()[$key];

                $zipReader = fopen($drawing->getPath(), 'r');
                $imageContents = '';
                while (!feof($zipReader)) {
                    $imageContents .= fread($zipReader, 1024);
                }
                fclose($zipReader);
                $extension = $drawing->getExtension();

                $imageSRC[$drawing->getCoordinates()] = "data:image/jpeg;base64," . base64_encode($imageContents);
            }
        }

        $import = new BulkImport($imageSRC);
        Excel::import($import, $request->file('file'));

        $excelError = $import->data['excel_error_key'];

        array_shift($excelError);
        $excelError = array_values($excelError);

        $errorMessage = "";
        if (!empty($excelError)) {
            $errorMessage = "In excel may be some issue or blank data at rows " . implode(',', $excelError);
        }

        return back()->with('message', 'Excel Data Imported successfully. '.$errorMessage);
    }
```

### BulkImport 

```
<?php

namespace App\Imports;

use App\Product;
use Illuminate\Support\Collection;
use Maatwebsite\Excel\Concerns\ToCollection;
use Auth;
use App\Category;
use App\ProductDetails;
use Image;

class BulkImport implements ToCollection {

    protected $productImages;

    public function __construct($productImages) {
        $this->productImages = $productImages;
    }

    public function collection(Collection $rows) {
        $firstAttr = 4;
        $lasrtAttr = $rows->first()->keys()->last();
        $headerRow = $rows->first()->toArray();

        $productIds = [];
        $invalidKey = [];

        $rows = $rows->toArray();

        $rows = array_combine(range(1, count($rows)), array_values($rows));

        foreach ($rows as $key => $row)
        {
            if (!in_array($key, [0, 1]) && isset($this->productImages['D' . $key])) {

                $getCategoryId = Category::where('user_id', Auth::user()->id)->where('name', $row[1])->first();

                if (!empty($getCategoryId) || !is_null($getCategoryId)) {
                    $pngUrl = $this->imageUpload($key);

                    $productCount = Product::where('user_id', Auth::user()->id)
                            ->where('category_id', $getCategoryId->id)
                            ->get()
                            ->count();

                    $product = Product::create([
                                'user_id' => Auth::user()->id,
                                'category_id' => $getCategoryId->id,
                                'product_name' => $row[2],
                                'image' => $pngUrl,
                                'position' => ($productCount + 1)
                    ]);

                    $productIds[] = $product->id;

                    $this->saveProductDetails($firstAttr, $lasrtAttr, $product, $headerRow, $row);
                }
            } else {
                $invalidKey[] = $key;
            }
        }

       // dd($productIds, $invalidKey);

        $this->data['product_ids'] = $productIds;
        $this->data['excel_error_key'] = $invalidKey;
    }

    public function imageUpload($key) {
        $pngUrl = time() . rand(1, 99999) . ".png";
        $pathThumbnail = public_path('/thumbnails');

        Image::make(file_get_contents($this->productImages['D' . $key]))->resize(100, 100, function ($constraint)
        {
            $constraint->aspectRatio();
        })->save($pathThumbnail . '/' . $pngUrl);

        $pathUpload = public_path('/upload');
        Image::make(file_get_contents($this->productImages['D' . $key]))->save("" . $pathUpload . "/" . $pngUrl, 60);

        return $pngUrl;
    }

    public function saveProductDetails($firstAttr, $lasrtAttr, $product, $headerRow, $row) {
        $productDetailData = array();
        for ($i = $firstAttr; $i <= $lasrtAttr; $i++)
        {
            array_push($productDetailData, array(
                'product_id' => $product->id,
                'meta_key' => $headerRow[$i],
                'meta_value' => isset($row[$i]) ? $row[$i] : ""
            ));
        }

        ProductDetails::insert($productDetailData);

        return true;
    }

}
```
