import React, { useState, useEffect, useRef, useMemo } from 'react';
import { 
  Upload, 
  Copy, 
  Check, 
  Download, 
  Search, 
  ChevronUp, 
  ChevronDown, 
  ChevronsUpDown,
  Smartphone,
  ShoppingBag,
  FileSpreadsheet,
  X,
  FileText,
  AlertCircle,
  Filter,
  CheckSquare,
  Square,
  Trash2,
  GitCompare,
  CheckCircle2,
  Clock,
  HelpCircle,
  ArrowRight
} from 'lucide-react';

function cleanFileDate(val) {
  if (val === null || val === undefined || val === "") return "-";

  // If already a JavaScript Date object
  if (val instanceof Date && !isNaN(val.getTime())) {
    const day = String(val.getDate()).padStart(2, '0');
    const month = String(val.getMonth() + 1).padStart(2, '0');
    const year = val.getFullYear();
    return `${day}/${month}/${year}`;
  }

  // If it's an Excel numeric serial date (e.g. 45564)
  const num = Number(val);
  if (!isNaN(num) && num > 20000 && num < 75000) {
    const totalMs = Math.round((num - 25569) * 86400 * 1000);
    const dateObj = new Date(totalMs);
    const userOffset = dateObj.getTimezoneOffset() * 60000;
    const adjusted = new Date(dateObj.getTime() + userOffset);
    const day = String(adjusted.getDate()).padStart(2, '0');
    const month = String(adjusted.getMonth() + 1).padStart(2, '0');
    const year = adjusted.getFullYear();
    return `${day}/${month}/${year}`;
  }

  let str = String(val).trim();
  if (!str || str === "-" || str === "NaN" || str === "null" || str === "undefined") return "-";

  // If ISO string like 2024-09-28T00:00:00.000Z
  if (str.includes("T")) {
    str = str.split("T")[0];
  }

  // Strip all variations of time (e.g. " 14:20:00", " 02:22 PM", " 12:00:00 AM", " 00:00:00")
  str = str.replace(/\s+\d{1,2}:\d{2}(:\d{2})?(\s*(AM|PM|am|pm))?.*/i, '').trim();

  // If format is YYYY-MM-DD, convert to clean DD/MM/YYYY
  const ymdMatch = str.match(/^(\d{4})[-/.](\d{1,2})[-/.](\d{1,2})$/);
  if (ymdMatch) {
    const y = ymdMatch[1];
    const m = String(ymdMatch[2]).padStart(2, '0');
    const d = String(ymdMatch[3]).padStart(2, '0');
    return `${d}/${m}/${y}`;
  }

  // If format is DD-MM-YYYY or DD/MM/YYYY, normalize to standard DD/MM/YYYY
  const dmyMatch = str.match(/^(\d{1,2})[-/.](\d{1,2})[-/.](\d{4})$/);
  if (dmyMatch) {
    const d = String(dmyMatch[1]).padStart(2, '0');
    const m = String(dmyMatch[2]).padStart(2, '0');
    const y = dmyMatch[3];
    return `${d}/${m}/${y}`;
  }

  return str || "-";
}

function getTimestampForSort(val) {
  if (!val || val === "-") return 0;
  const str = String(val).trim();

  const dmyMatch = str.match(/^(\d{1,2})[-/.](\d{1,2})[-/.](\d{4})/);
  if (dmyMatch) {
    const day = parseInt(dmyMatch[1], 10);
    const month = parseInt(dmyMatch[2], 10) - 1;
    const year = parseInt(dmyMatch[3], 10);
    return new Date(year, month, day).getTime();
  }

  const ymdMatch = str.match(/^(\d{4})[-/.](\d{1,2})[-/.](\d{1,2})/);
  if (ymdMatch) {
    const year = parseInt(ymdMatch[1], 10);
    const month = parseInt(ymdMatch[2], 10) - 1;
    const day = parseInt(ymdMatch[3], 10);
    return new Date(year, month, day).getTime();
  }

  const parsed = Date.parse(str);
  return isNaN(parsed) ? 0 : parsed;
}

function getNumericValue(val) {
  if (val === null || val === undefined) return 0;
  const cleaned = String(val).replace(/[^0-9.-]/g, '');
  const num = parseFloat(cleaned);
  return isNaN(num) ? 0 : num;
}

function normalizeKey(str) {
  if (!str) return "";
  return String(str).trim().replace(/[^a-zA-Z0-9]/g, '').toUpperCase();
}

async function copyTextToClipboard(text) {
  if (!text) return false;

  // Try modern Clipboard API if supported and permitted
  if (navigator.clipboard && typeof navigator.clipboard.writeText === "function") {
    try {
      await navigator.clipboard.writeText(text);
      return true;
    } catch {
      // Permissions policy or focus blocked, proceed to fallback
    }
  }

  // Fallback using document.execCommand('copy') for sandboxed iframes
  try {
    const textArea = document.createElement("textarea");
    textArea.value = text;
    textArea.style.position = "fixed";
    textArea.style.top = "-9999px";
    textArea.style.left = "-9999px";
    textArea.style.opacity = "0";
    textArea.setAttribute("readonly", "");
    document.body.appendChild(textArea);
    textArea.focus();
    textArea.select();
    const successful = document.execCommand("copy");
    document.body.removeChild(textArea);
    return successful;
  } catch (err) {
    console.warn("Clipboard fallback failed:", err);
    return false;
  }
}

export default function App() {
  const [activeTab, setActiveTab] = useState("imei_activation");

  // Tab 1: IMEI Activation State
  const [imeiData, setImeiData] = useState([]);
  const [imeiFileName, setImeiFileName] = useState("");
  const [imeiSearch, setImeiSearch] = useState("");
  const [imeiSortField, setImeiSortField] = useState("activationDate");
  const [imeiSortOrder, setImeiSortOrder] = useState("desc");
  const [imeiPage, setImeiPage] = useState(1);
  const [copiedImei, setCopiedImei] = useState(null);

  // Tab 2: Product-Wise Sales Report State
  const [rawSalesData, setRawSalesData] = useState([]);
  const [salesFileName, setSalesFileName] = useState("");
  const [salesSearch, setSalesSearch] = useState("");
  const [selectedProductTypes, setSelectedProductTypes] = useState(new Set());
  const [isTypeModalOpen, setIsTypeModalOpen] = useState(false);
  const [modalTempSelected, setModalTempSelected] = useState(new Set());
  const [salesSortField, setSalesSortField] = useState("productCode");
  const [salesSortOrder, setSalesSortOrder] = useState("asc");
  const [salesPage, setSalesPage] = useState(1);
  const [copiedSerial, setCopiedSerial] = useState(null);
  const [salesUploadError, setSalesUploadError] = useState("");

  // Tab 3: EPOS Reconciliation State
  const [reconFilter, setReconFilter] = useState("all"); // 'all' | 'activated' | 'pending' | 'not_in_pos'
  const [reconSearch, setReconSearch] = useState("");
  const [reconPage, setReconPage] = useState(1);
  const [copiedPending, setCopiedPending] = useState(false);

  // Clear modal state
  const [isClearModalOpen, setIsClearModalOpen] = useState(false);

  const hasCurrentData = activeTab === "imei_activation" 
    ? imeiData.length > 0 
    : activeTab === "sales_report"
    ? rawSalesData.length > 0
    : (imeiData.length > 0 || rawSalesData.length > 0);

  const handleConfirmClear = () => {
    if (activeTab === "imei_activation") {
      setImeiData([]);
      setImeiFileName("");
      setImeiSearch("");
      setImeiPage(1);
      if (imeiFileInputRef.current) imeiFileInputRef.current.value = "";
    } else if (activeTab === "sales_report") {
      setRawSalesData([]);
      setSalesFileName("");
      setSalesSearch("");
      setSelectedProductTypes(new Set());
      setModalTempSelected(new Set());
      setSalesPage(1);
      setSalesUploadError("");
      if (salesFileInputRef.current) salesFileInputRef.current.value = "";
    } else {
      // Clear both if cleared from EPOS Reconciliation tab
      setImeiData([]);
      setImeiFileName("");
      setRawSalesData([]);
      setSalesFileName("");
      setSelectedProductTypes(new Set());
      setReconPage(1);
      if (imeiFileInputRef.current) imeiFileInputRef.current.value = "";
      if (salesFileInputRef.current) salesFileInputRef.current.value = "";
    }
    setIsClearModalOpen(false);
  };

  const rowsPerPage = 50;
  const imeiFileInputRef = useRef(null);
  const salesFileInputRef = useRef(null);

  useEffect(() => {
    if (!window.XLSX) {
      const script = document.createElement("script");
      script.src = "https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js";
      script.async = true;
      document.body.appendChild(script);
    }
  }, []);

  const readWorkbookFromFile = (file) => {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = (e) => {
        try {
          if (!window.XLSX) {
            reject(new Error("Spreadsheet engine is initializing, please try again."));
            return;
          }
          const data = new Uint8Array(e.target.result);
          const workbook = window.XLSX.read(data, { 
            type: 'array',
            cellDates: true,
            cellNF: true,
            cellText: true
          });
          resolve(workbook);
        } catch (err) {
          reject(err);
        }
      };
      reader.onerror = reject;
      reader.readAsArrayBuffer(file);
    });
  };

  const handleImeiUpload = async (file) => {
    if (!file) return;
    setImeiFileName(file.name);

    try {
      const workbook = await readWorkbookFromFile(file);
      const firstSheet = workbook.SheetNames[0];
      const worksheet = workbook.Sheets[firstSheet];
      
      const rawMatrix = window.XLSX.utils.sheet_to_json(worksheet, { header: 1, defval: "" });
      if (!rawMatrix || rawMatrix.length === 0) return;

      // Locate the row where data actually begins (skip initial sheet title / blank rows)
      let startRow = 0;
      for (let i = 0; i < Math.min(15, rawMatrix.length); i++) {
        const row = rawMatrix[i] || [];
        const cellA = String(row[0] || "").trim().toLowerCase();
        const cellB = String(row[1] || "").trim().toLowerCase();
        if (cellA.includes("imei") || cellB.includes("activation") || cellB.includes("date")) {
          startRow = i + 1; // Real data begins on the very next line after header
          break;
        }
      }

      // Column AF in 0-based index:
      // A=0, B=1, C=2, D=3, E=4... Z=25, AA=26, AB=27, AC=28, AD=29, AE=30, AF=31
      const COL_A_IMEI = 0;
      const COL_B_DATE = 1;
      const COL_C_MODEL = 2;
      const COL_D_CODE = 3;
      const COL_AF_MRP = 31;

      const parsed = [];
      for (let r = startRow; r < rawMatrix.length; r++) {
        const row = rawMatrix[r];
        if (!row || row.length === 0) continue;

        // Strictly take Column A: IMEI
        const imeiRaw = row[COL_A_IMEI];
        const imei = String(imeiRaw !== undefined && imeiRaw !== null ? imeiRaw : "").trim();

        // Skip blank lines, header repeats, or total footers
        if (!imei || imei.toLowerCase() === "imei" || imei.toLowerCase().includes("total") || imei === "-") {
          continue;
        }

        // Strictly take Column B: ActivationDate from cell or formatted cell
        // Check raw row array, plus cell formatted text (.w) if present on worksheet
        let rawDate = row[COL_B_DATE];
        const cellCoord = window.XLSX.utils.encode_cell({ r, c: COL_B_DATE });
        const cellRef = worksheet ? worksheet[cellCoord] : null;
        if (cellRef && cellRef.w && !cellRef.w.includes("#")) {
          rawDate = cellRef.w;
        }

        const activationDate = cleanFileDate(rawDate);

        // Strictly take Column C: ModelNo
        const rawModel = row[COL_C_MODEL];
        const modelNo = String(rawModel !== undefined && rawModel !== null ? rawModel : "").trim() || "-";

        // Strictly take Column D: ProductCode
        const rawProdCode = row[COL_D_CODE];
        const productCode = String(rawProdCode !== undefined && rawProdCode !== null ? rawProdCode : "").trim() || "-";

        // Strictly take Column AF: MRP (index 31)
        let rawMrp = row[COL_AF_MRP];
        const mrpCellCoord = window.XLSX.utils.encode_cell({ r, c: COL_AF_MRP });
        const mrpCellRef = worksheet ? worksheet[mrpCellCoord] : null;
        if (mrpCellRef && mrpCellRef.w) {
          rawMrp = mrpCellRef.w;
        }
        const mrp = String(rawMrp !== undefined && rawMrp !== null ? rawMrp : "").trim() || "-";

        parsed.push({
          id: parsed.length + 1,
          imei,
          activationDate,
          modelNo,
          productCode,
          mrp
        });
      }

      setImeiData(parsed);
      setImeiPage(1);
    } catch (err) {
      console.error("IMEI Parsing Error:", err);
    }
  };

  const handleSalesUpload = async (file) => {
    if (!file) return;
    setSalesFileName(file.name);
    setSalesUploadError("");

    try {
      const workbook = await readWorkbookFromFile(file);
      const firstSheet = workbook.SheetNames[0];
      const worksheet = workbook.Sheets[firstSheet];

      const rawMatrix = window.XLSX.utils.sheet_to_json(worksheet, { header: 1, defval: "" });
      
      if (!rawMatrix || rawMatrix.length === 0) {
        setSalesUploadError("The uploaded file is empty.");
        return;
      }

      let headerRowIndex = 0;
      let maxMatchCount = 0;

      for (let r = 0; r < Math.min(25, rawMatrix.length); r++) {
        const row = rawMatrix[r];
        if (!Array.isArray(row)) continue;

        const rowJoined = row
          .map(cell => String(cell).toLowerCase().replace(/[^a-z0-9]/g, ''))
          .join(" ");

        let matches = 0;
        if (rowJoined.includes("producttype") || rowJoined.includes("itemtype") || rowJoined.includes("type")) matches++;
        if (rowJoined.includes("productcode") || rowJoined.includes("itemcode") || rowJoined.includes("code") || rowJoined.includes("sku")) matches++;
        if (rowJoined.includes("serial") || rowJoined.includes("imei") || rowJoined.includes("srno")) matches++;

        if (matches > maxMatchCount) {
          maxMatchCount = matches;
          headerRowIndex = r;
        }
      }

      const headers = rawMatrix[headerRowIndex].map(h => 
        String(h || "").trim().toLowerCase().replace(/[^a-z0-9]/g, '')
      );

      const findHeaderIndex = (keywords) => {
        let idx = headers.findIndex(h => keywords.some(k => h === k));
        if (idx !== -1) return idx;
        idx = headers.findIndex(h => keywords.some(k => h.includes(k)));
        return idx;
      };

      const typeIdx = findHeaderIndex(["producttype", "itemtype", "type"]);
      const codeIdx = findHeaderIndex(["productcode", "itemcode", "code", "materialcode", "sku", "barcode"]);
      const serialIdx = findHeaderIndex(["serialnumber", "serialno", "serial", "serials", "srno", "imei", "deviceid"]);

      const parsedRows = [];
      const detectedTypes = new Set();

      for (let r = headerRowIndex + 1; r < rawMatrix.length; r++) {
        const row = rawMatrix[r];
        if (!row || row.length === 0) continue;

        const firstCell = String(row[0] || "").toLowerCase();
        if (firstCell.includes("total") || firstCell.includes("grand total")) continue;

        const productType = typeIdx !== -1 && row[typeIdx] !== undefined ? String(row[typeIdx]).trim() : "-";
        const productCode = codeIdx !== -1 && row[codeIdx] !== undefined ? String(row[codeIdx]).trim() : "-";
        const serialNumber = serialIdx !== -1 && row[serialIdx] !== undefined ? String(row[serialIdx]).trim() : "-";

        const hasContent = (productCode && productCode !== "-") || 
                           (serialNumber && serialNumber !== "-") || 
                           (productType && productType !== "-");

        if (hasContent) {
          parsedRows.push({
            id: parsedRows.length + 1,
            productType: productType || "-",
            productCode: productCode || "-",
            serialNumber: serialNumber || "-"
          });

          if (productType && productType !== "-") {
            detectedTypes.add(productType);
          }
        }
      }

      if (parsedRows.length === 0) {
        setSalesUploadError("No data rows found. Please check columns in your file.");
      } else {
        setRawSalesData(parsedRows);
        setSelectedProductTypes(new Set(detectedTypes));
        setModalTempSelected(new Set(detectedTypes));
        setIsTypeModalOpen(true);
        setSalesPage(1);
      }
    } catch (err) {
      console.error("Sales Report Parsing Error:", err);
      setSalesUploadError("Error reading file: " + (err.message || "Invalid format"));
    }
  };

  const copyToClipboard = async (text, setter) => {
    await copyTextToClipboard(text);
    setter(text);
    setTimeout(() => setter(null), 1500);
  };

  const copyAllIMEIs = async () => {
    const list = processedImeiData.map(d => d.imei).join("\n");
    await copyTextToClipboard(list);
    setCopiedImei("ALL");
    setTimeout(() => setCopiedImei(null), 1500);
  };

  const copyAllSerials = async () => {
    const list = processedSalesData.map(d => d.serialNumber).filter(s => s && s !== "-").join("\n");
    await copyTextToClipboard(list);
    setCopiedSerial("ALL");
    setTimeout(() => setCopiedSerial(null), 1500);
  };

  const exportCleanImeiCSV = () => {
    const headers = ["IMEI", "ActivationDate", "ModelNo", "ProductCode", "MRP"];
    const rows = processedImeiData.map(d => [d.imei, d.activationDate, d.modelNo, d.productCode, d.mrp]);
    const csvContent = "data:text/csv;charset=utf-8," 
      + [headers.join(","), ...rows.map(r => r.map(v => `"${String(v).replace(/"/g, '""')}"`).join(","))].join("\n");
    
    const encodedUri = encodeURI(csvContent);
    const link = document.createElement("a");
    link.setAttribute("href", encodedUri);
    link.setAttribute("download", `${(imeiFileName || "IMEI_Cleaned").replace(/\.[^/.]+$/, "")}.csv`);
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  };

  const exportCleanSalesCSV = () => {
    const headers = ["ProductType", "ProductCode", "SerialNumber"];
    const rows = processedSalesData.map(d => [d.productType, d.productCode, d.serialNumber]);
    const csvContent = "data:text/csv;charset=utf-8," 
      + [headers.join(","), ...rows.map(r => r.map(v => `"${String(v).replace(/"/g, '""')}"`).join(","))].join("\n");
    
    const encodedUri = encodeURI(csvContent);
    const link = document.createElement("a");
    link.setAttribute("href", encodedUri);
    link.setAttribute("download", `${(salesFileName || "SalesReport_Clean").replace(/\.[^/.]+$/, "")}_clean.csv`);
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  };

  const handleImeiSort = (field) => {
    if (imeiSortField === field) {
      setImeiSortOrder(prev => (prev === "asc" ? "desc" : "asc"));
    } else {
      setImeiSortField(field);
      setImeiSortOrder(field === "activationDate" || field === "mrp" ? "desc" : "asc");
    }
  };

  const handleSalesSort = (field) => {
    if (salesSortField === field) {
      setSalesSortOrder(prev => (prev === "asc" ? "desc" : "asc"));
    } else {
      setSalesSortField(field);
      setSalesSortOrder("asc");
    }
  };

  const productTypeStats = useMemo(() => {
    const counts = {};
    rawSalesData.forEach(item => {
      const type = item.productType || "-";
      counts[type] = (counts[type] || 0) + 1;
    });

    return Object.keys(counts)
      .filter(t => t !== "-")
      .sort()
      .map(type => ({
        type,
        count: counts[type]
      }));
  }, [rawSalesData]);

  const processedImeiData = useMemo(() => {
    const filtered = imeiData.filter(item => {
      if (!imeiSearch) return true;
      const q = imeiSearch.toLowerCase();
      return (
        item.imei.toLowerCase().includes(q) ||
        item.activationDate.toLowerCase().includes(q) ||
        item.modelNo.toLowerCase().includes(q) ||
        item.productCode.toLowerCase().includes(q) ||
        item.mrp.toLowerCase().includes(q)
      );
    });

    return [...filtered].sort((a, b) => {
      let comparison = 0;
      if (imeiSortField === "activationDate") {
        comparison = getTimestampForSort(a.activationDate) - getTimestampForSort(b.activationDate);
      } else if (imeiSortField === "mrp") {
        comparison = getNumericValue(a.mrp) - getNumericValue(b.mrp);
      } else if (imeiSortField === "imei") {
        comparison = a.imei.localeCompare(b.imei, undefined, { numeric: true });
      } else {
        comparison = (a[imeiSortField] || "").toString().localeCompare((b[imeiSortField] || "").toString());
      }
      return imeiSortOrder === "asc" ? comparison : -comparison;
    });
  }, [imeiData, imeiSearch, imeiSortField, imeiSortOrder]);

  const processedSalesData = useMemo(() => {
    if (!rawSalesData.length) return [];
    
    const filtered = rawSalesData.filter(row => {
      if (selectedProductTypes.size > 0 && !selectedProductTypes.has(row.productType)) {
        return false;
      }

      if (!salesSearch) return true;
      const q = salesSearch.toLowerCase();
      return (
        row.productType.toLowerCase().includes(q) ||
        row.productCode.toLowerCase().includes(q) ||
        row.serialNumber.toLowerCase().includes(q)
      );
    });

    return [...filtered].sort((a, b) => {
      const valA = (a[salesSortField] || "").toString();
      const valB = (b[salesSortField] || "").toString();
      const comparison = valA.localeCompare(valB, undefined, { numeric: true });
      return salesSortOrder === "asc" ? comparison : -comparison;
    });
  }, [rawSalesData, salesSearch, selectedProductTypes, salesSortField, salesSortOrder]);

  const reconciliationData = useMemo(() => {
    if (!imeiData.length && !rawSalesData.length) {
      return {
        items: [],
        counts: { totalSold: 0, totalActivated: 0, matched: 0, pending: 0, notInPos: 0 },
        rate: 0
      };
    }

    const imeiMap = new Map();
    imeiData.forEach(item => {
      const norm = normalizeKey(item.imei);
      if (norm) {
        imeiMap.set(norm, item);
      }
    });

    const matchedSalesNorms = new Set();
    const rows = [];

    // Filter relevant sales by selected product types if filtered, else all sales
    const activeSalesList = rawSalesData.filter(s => 
      selectedProductTypes.size === 0 || selectedProductTypes.has(s.productType)
    );

    // 1. Check each sale against IMEI activations
    activeSalesList.forEach(sale => {
      const serialNorm = normalizeKey(sale.serialNumber);
      const isCleanSerial = serialNorm && serialNorm !== "-" && serialNorm.length >= 6;

      if (isCleanSerial && imeiMap.has(serialNorm)) {
        const imeiMatch = imeiMap.get(serialNorm);
        matchedSalesNorms.add(serialNorm);
        rows.push({
          id: `match_${rows.length}`,
          identifier: sale.serialNumber || imeiMatch.imei,
          productType: sale.productType || "-",
          productCode: sale.productCode || imeiMatch.productCode || "-",
          modelNo: imeiMatch.modelNo || "-",
          mrp: imeiMatch.mrp || "-",
          activationDate: imeiMatch.activationDate || "-",
          status: "activated" // Sold & Activated
        });
      } else {
        rows.push({
          id: `pending_${rows.length}`,
          identifier: sale.serialNumber || "-",
          productType: sale.productType || "-",
          productCode: sale.productCode || "-",
          modelNo: "-",
          mrp: "-",
          activationDate: "-",
          status: "pending" // Sold in POS, Pending Activation
        });
      }
    });

    // 2. Identify IMEIs activated in carrier report but not present in POS sales
    imeiData.forEach(imeiItem => {
      const imeiNorm = normalizeKey(imeiItem.imei);
      if (imeiNorm && !matchedSalesNorms.has(imeiNorm)) {
        rows.push({
          id: `notinpos_${rows.length}`,
          identifier: imeiItem.imei,
          productType: "-",
          productCode: imeiItem.productCode || "-",
          modelNo: imeiItem.modelNo || "-",
          mrp: imeiItem.mrp || "-",
          activationDate: imeiItem.activationDate || "-",
          status: "not_in_pos" // In IMEI report but missing in POS sales
        });
      }
    });

    const totalSold = activeSalesList.length;
    const totalActivated = imeiData.length;
    const matched = matchedSalesNorms.size;
    const pending = Math.max(0, totalSold - matched);
    const notInPos = Math.max(0, totalActivated - matched);
    const rate = totalSold > 0 ? Math.round((matched / totalSold) * 100) : 0;

    return {
      items: rows,
      counts: { totalSold, totalActivated, matched, pending, notInPos },
      rate
    };
  }, [imeiData, rawSalesData, selectedProductTypes]);

  const filteredReconData = useMemo(() => {
    let list = reconciliationData.items;

    if (reconFilter !== "all") {
      list = list.filter(r => r.status === reconFilter);
    }

    if (reconSearch.trim()) {
      const q = reconSearch.toLowerCase().trim();
      list = list.filter(r => 
        r.identifier.toLowerCase().includes(q) ||
        r.productCode.toLowerCase().includes(q) ||
        r.productType.toLowerCase().includes(q) ||
        r.modelNo.toLowerCase().includes(q)
      );
    }

    return list;
  }, [reconciliationData.items, reconFilter, reconSearch]);

  const paginatedReconData = useMemo(() => {
    const start = (reconPage - 1) * rowsPerPage;
    return filteredReconData.slice(start, start + rowsPerPage);
  }, [filteredReconData, reconPage]);

  const reconTotalPages = Math.ceil(filteredReconData.length / rowsPerPage) || 1;

  const copyPendingSerials = async () => {
    const pendingList = reconciliationData.items
      .filter(i => i.status === "pending" && i.identifier && i.identifier !== "-")
      .map(i => i.identifier)
      .join("\n");

    if (pendingList) {
      await copyTextToClipboard(pendingList);
      setCopiedPending(true);
      setTimeout(() => setCopiedPending(false), 1500);
    }
  };

  const exportReconciliationCSV = () => {
    const headers = ["Status", "Serial_or_IMEI", "ProductType", "ProductCode", "ModelNo", "ActivationDate", "MRP"];
    const rows = filteredReconData.map(r => [
      r.status === "activated" ? "Activated" : r.status === "pending" ? "Pending Activation" : "Not in POS",
      r.identifier,
      r.productType,
      r.productCode,
      r.modelNo,
      r.activationDate,
      r.mrp
    ]);

    const csvContent = "data:text/csv;charset=utf-8," 
      + [headers.join(","), ...rows.map(row => row.map(v => `"${String(v).replace(/"/g, '""')}"`).join(","))].join("\n");
    
    const encodedUri = encodeURI(csvContent);
    const link = document.createElement("a");
    link.setAttribute("href", encodedUri);
    link.setAttribute("download", `EPOS_Reconciliation_Report.csv`);
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  };

  const paginatedImeiData = useMemo(() => {
    const start = (imeiPage - 1) * rowsPerPage;
    return processedImeiData.slice(start, start + rowsPerPage);
  }, [processedImeiData, imeiPage]);

  const imeiTotalPages = Math.ceil(processedImeiData.length / rowsPerPage) || 1;

  const paginatedSalesData = useMemo(() => {
    const start = (salesPage - 1) * rowsPerPage;
    return processedSalesData.slice(start, start + rowsPerPage);
  }, [processedSalesData, salesPage]);

  const salesTotalPages = Math.ceil(processedSalesData.length / rowsPerPage) || 1;

  const openProductTypeModal = () => {
    setModalTempSelected(new Set(selectedProductTypes));
    setIsTypeModalOpen(true);
  };

  const toggleModalItem = (type) => {
    setModalTempSelected(prev => {
      const next = new Set(prev);
      if (next.has(type)) {
        next.delete(type);
      } else {
        next.add(type);
      }
      return next;
    });
  };

  const selectAllModalTypes = () => {
    const all = new Set(productTypeStats.map(s => s.type));
    setModalTempSelected(all);
  };

  const deselectAllModalTypes = () => {
    setModalTempSelected(new Set());
  };

  const applyModalSelection = () => {
    setSelectedProductTypes(new Set(modalTempSelected));
    setIsTypeModalOpen(false);
    setSalesPage(1);
  };

  const renderSortIndicator = (field, currentField, currentOrder) => {
    if (currentField !== field) {
      return <ChevronsUpDown className="w-3.5 h-3.5 text-zinc-600 group-hover:text-zinc-400 ml-1 inline" />;
    }
    return currentOrder === "asc" ? (
      <ChevronUp className="w-3.5 h-3.5 text-zinc-200 ml-1 inline" />
    ) : (
      <ChevronDown className="w-3.5 h-3.5 text-zinc-200 ml-1 inline" />
    );
  };

  return (
    <div className="min-h-screen bg-[#090a0f] text-zinc-200 font-sans antialiased pb-24 selection:bg-zinc-800">
      
      {/* Top Application Bar */}
      <header className="h-14 border-b border-zinc-800/80 bg-[#0d0f17]/90 backdrop-blur px-6 flex items-center justify-between sticky top-0 z-30">
        <div className="flex items-center space-x-3">
          <div className="w-8 h-8 rounded-md bg-zinc-800 border border-zinc-700/60 flex items-center justify-center text-zinc-200">
            {activeTab === "imei_activation" ? (
              <Smartphone className="w-4 h-4" />
            ) : activeTab === "sales_report" ? (
              <ShoppingBag className="w-4 h-4" />
            ) : (
              <GitCompare className="w-4 h-4 text-emerald-400" />
            )}
          </div>
          <div className="flex items-center space-x-2">
            <span className="font-semibold text-sm tracking-tight text-white">
              {activeTab === "imei_activation" 
                ? "IMEI Activation" 
                : activeTab === "sales_report" 
                ? "Product Sales" 
                : "EPOS Reconciliation"}
            </span>
            {activeTab === "imei_activation" && imeiFileName && (
              <span className="text-xs text-zinc-400 font-mono px-2 py-0.5 rounded bg-zinc-800/70 border border-zinc-700/50">
                {imeiFileName}
              </span>
            )}
            {activeTab === "sales_report" && salesFileName && (
              <span className="text-xs text-zinc-400 font-mono px-2 py-0.5 rounded bg-zinc-800/70 border border-zinc-700/50">
                {salesFileName}
              </span>
            )}
            {activeTab === "epos_reconciliation" && (
              <span className="text-xs text-zinc-400 font-mono px-2 py-0.5 rounded bg-zinc-800/70 border border-zinc-700/50">
                Cross-Check Console
              </span>
            )}
          </div>
        </div>

        {/* Header Actions */}
        <div className="flex items-center space-x-2">
          {hasCurrentData && (
            <button
              onClick={() => setIsClearModalOpen(true)}
              className="h-8 px-2.5 rounded-md bg-zinc-900 hover:bg-red-950/40 border border-zinc-800 hover:border-red-800/60 text-zinc-400 hover:text-red-300 text-xs font-medium inline-flex items-center space-x-1.5 transition active:scale-[0.98]"
              title="Clear data"
            >
              <Trash2 className="w-3.5 h-3.5" />
              <span>Clear All</span>
            </button>
          )}

          {activeTab === "imei_activation" && (
            <>
              <input 
                type="file" 
                ref={imeiFileInputRef} 
                className="hidden" 
                accept=".xls,.xlsx,.csv" 
                onChange={(e) => {
                  if (e.target.files && e.target.files[0]) {
                    handleImeiUpload(e.target.files[0]);
                  }
                }} 
              />
              <button
                onClick={() => imeiFileInputRef.current?.click()}
                className="h-8 px-3 rounded-md bg-zinc-100 hover:bg-white text-zinc-900 text-xs font-medium inline-flex items-center space-x-1.5 transition active:scale-[0.98]"
              >
                <Upload className="w-3.5 h-3.5" />
                <span>Upload IMEI Report</span>
              </button>
            </>
          )}

          {activeTab === "sales_report" && (
            <>
              <input 
                type="file" 
                ref={salesFileInputRef} 
                className="hidden" 
                accept=".csv,.xls,.xlsx" 
                onChange={(e) => {
                  if (e.target.files && e.target.files[0]) {
                    handleSalesUpload(e.target.files[0]);
                  }
                }} 
              />
              <button
                onClick={() => salesFileInputRef.current?.click()}
                className="h-8 px-3 rounded-md bg-zinc-100 hover:bg-white text-zinc-900 text-xs font-medium inline-flex items-center space-x-1.5 transition active:scale-[0.98]"
              >
                <Upload className="w-3.5 h-3.5" />
                <span>Upload Sales File</span>
              </button>
            </>
          )}

          {activeTab === "epos_reconciliation" && (
            <button
              onClick={exportReconciliationCSV}
              disabled={filteredReconData.length === 0}
              className="h-8 px-3 rounded-md bg-zinc-100 hover:bg-white disabled:opacity-40 disabled:pointer-events-none text-zinc-900 text-xs font-medium inline-flex items-center space-x-1.5 transition active:scale-[0.98]"
            >
              <Download className="w-3.5 h-3.5" />
              <span>Export Report</span>
            </button>
          )}
        </div>
      </header>

      {/* Main Content Area */}
      <main className="max-w-7xl mx-auto px-4 sm:px-6 py-6">
        
        {/* TAB 1: IMEI ACTIVATION VIEW */}
        {activeTab === "imei_activation" && (
          <div className="space-y-4">
            {imeiData.length === 0 ? (
              <div 
                onClick={() => imeiFileInputRef.current?.click()}
                className="cursor-pointer border border-dashed border-zinc-800 hover:border-zinc-700 rounded-xl p-16 text-center bg-[#0d0f17]/40 hover:bg-[#0d0f17]/70 transition flex flex-col items-center justify-center space-y-3"
              >
                <div className="w-10 h-10 rounded-lg bg-zinc-800/80 border border-zinc-700/60 flex items-center justify-center text-zinc-400">
                  <FileSpreadsheet className="w-5 h-5" />
                </div>
                <div className="space-y-1">
                  <p className="text-sm font-medium text-zinc-200">Import IMEI Activation Report</p>
                  <p className="text-xs text-zinc-500">Supports .xls, .xlsx, and .csv files</p>
                </div>
              </div>
            ) : (
              <>
                <div className="flex flex-col sm:flex-row items-stretch sm:items-center justify-between gap-3">
                  <div className="relative w-full sm:w-80">
                    <Search className="w-3.5 h-3.5 absolute left-3 top-2.5 text-zinc-500" />
                    <input
                      type="text"
                      value={imeiSearch}
                      onChange={(e) => {
                        setImeiSearch(e.target.value);
                        setImeiPage(1);
                      }}
                      placeholder="Search records..."
                      className="w-full bg-[#0d0f17] border border-zinc-800 rounded-md pl-8 pr-7 py-1.5 text-xs text-zinc-200 placeholder-zinc-500 focus:outline-none focus:border-zinc-600 transition"
                    />
                    {imeiSearch && (
                      <button
                        onClick={() => setImeiSearch("")}
                        className="absolute right-2.5 top-2 text-zinc-500 hover:text-zinc-300"
                      >
                        <X className="w-3.5 h-3.5" />
                      </button>
                    )}
                  </div>

                  <div className="flex items-center space-x-2 self-end sm:self-auto">
                    <button
                      onClick={copyAllIMEIs}
                      className="h-8 px-3 rounded-md bg-zinc-900 hover:bg-zinc-800 border border-zinc-800 text-xs text-zinc-300 inline-flex items-center space-x-1.5 transition"
                    >
                      {copiedImei === "ALL" ? <Check className="w-3.5 h-3.5 text-emerald-400" /> : <Copy className="w-3.5 h-3.5 text-zinc-400" />}
                      <span>{copiedImei === "ALL" ? "Copied" : "Copy All"}</span>
                    </button>

                    <button
                      onClick={exportCleanImeiCSV}
                      className="h-8 px-3 rounded-md bg-zinc-900 hover:bg-zinc-800 border border-zinc-800 text-xs text-zinc-300 inline-flex items-center space-x-1.5 transition"
                    >
                      <Download className="w-3.5 h-3.5 text-zinc-400" />
                      <span>Export CSV</span>
                    </button>
                  </div>
                </div>

                <div className="rounded-lg border border-zinc-800 bg-[#0d0f17] overflow-hidden">
                  <div className="overflow-x-auto">
                    <table className="w-full text-left border-collapse text-xs">
                      <thead>
                        <tr className="border-b border-zinc-800 text-zinc-400 bg-zinc-900/50">
                          <th className="py-2.5 px-3 font-medium text-zinc-500 w-12 text-center">#</th>
                          
                          <th 
                            onClick={() => handleImeiSort("imei")}
                            className="py-2.5 px-3 font-medium cursor-pointer hover:text-zinc-200 transition group select-none"
                          >
                            <span className="inline-flex items-center">
                              IMEI {renderSortIndicator("imei", imeiSortField, imeiSortOrder)}
                            </span>
                          </th>

                          <th 
                            onClick={() => handleImeiSort("activationDate")}
                            className="py-2.5 px-3 font-medium cursor-pointer hover:text-zinc-200 transition group select-none"
                          >
                            <span className="inline-flex items-center">
                              Activation Date {renderSortIndicator("activationDate", imeiSortField, imeiSortOrder)}
                            </span>
                          </th>

                          <th 
                            onClick={() => handleImeiSort("modelNo")}
                            className="py-2.5 px-3 font-medium cursor-pointer hover:text-zinc-200 transition group select-none"
                          >
                            <span className="inline-flex items-center">
                              Model No {renderSortIndicator("modelNo", imeiSortField, imeiSortOrder)}
                            </span>
                          </th>

                          <th 
                            onClick={() => handleImeiSort("productCode")}
                            className="py-2.5 px-3 font-medium cursor-pointer hover:text-zinc-200 transition group select-none"
                          >
                            <span className="inline-flex items-center">
                              Product Code {renderSortIndicator("productCode", imeiSortField, imeiSortOrder)}
                            </span>
                          </th>

                          <th 
                            onClick={() => handleImeiSort("mrp")}
                            className="py-2.5 px-3 font-medium text-right cursor-pointer hover:text-zinc-200 transition group select-none"
                          >
                            <span className="inline-flex items-center justify-end">
                              MRP {renderSortIndicator("mrp", imeiSortField, imeiSortOrder)}
                            </span>
                          </th>
                        </tr>
                      </thead>

                      <tbody className="divide-y divide-zinc-800/60 font-mono text-[11px]">
                        {paginatedImeiData.length === 0 ? (
                          <tr>
                            <td colSpan="6" className="py-10 text-center text-zinc-500 font-sans">
                              No records match your search.
                            </td>
                          </tr>
                        ) : (
                          paginatedImeiData.map((item, index) => (
                            <tr key={item.id} className="hover:bg-zinc-800/30 transition-colors">
                              <td className="py-2 px-3 text-center text-zinc-600">
                                {(imeiPage - 1) * rowsPerPage + index + 1}
                              </td>

                              <td className="py-2 px-3 text-zinc-200">
                                <div className="flex items-center space-x-2">
                                  <span>{item.imei}</span>
                                  <button
                                    onClick={() => copyToClipboard(item.imei, setCopiedImei)}
                                    className="p-1 rounded text-zinc-500 hover:text-zinc-300 transition"
                                  >
                                    {copiedImei === item.imei ? (
                                      <Check className="w-3 h-3 text-emerald-400" />
                                    ) : (
                                      <Copy className="w-3 h-3" />
                                    )}
                                  </button>
                                </div>
                              </td>

                              <td className="py-2 px-3 text-zinc-300">
                                {item.activationDate}
                              </td>

                              <td className="py-2 px-3 text-zinc-400 font-sans">
                                {item.modelNo}
                              </td>

                              <td className="py-2 px-3 text-zinc-400">
                                {item.productCode}
                              </td>

                              <td className="py-2 px-3 text-right text-zinc-200">
                                {item.mrp.startsWith("₹") ? item.mrp : `₹${item.mrp}`}
                              </td>
                            </tr>
                          ))
                        )}
                      </tbody>
                    </table>
                  </div>

                  <div className="h-10 px-4 border-t border-zinc-800 bg-zinc-900/30 flex items-center justify-between text-xs text-zinc-500">
                    <div>
                      {processedImeiData.length > 0 ? (
                        <span>
                          {(imeiPage - 1) * rowsPerPage + 1}–{Math.min(imeiPage * rowsPerPage, processedImeiData.length)} of {processedImeiData.length}
                        </span>
                      ) : (
                        <span>0 records</span>
                      )}
                    </div>

                    <div className="flex items-center space-x-1">
                      <button
                        disabled={imeiPage === 1}
                        onClick={() => setImeiPage(p => Math.max(1, p - 1))}
                        className="px-2 py-1 rounded bg-zinc-800/50 hover:bg-zinc-800 disabled:opacity-30 disabled:cursor-not-allowed text-zinc-300 text-[11px]"
                      >
                        Prev
                      </button>
                      <span className="px-2 text-zinc-400 text-[11px]">
                        {imeiPage} / {imeiTotalPages}
                      </span>
                      <button
                        disabled={imeiPage >= imeiTotalPages}
                        onClick={() => setImeiPage(p => Math.min(imeiTotalPages, p + 1))}
                        className="px-2 py-1 rounded bg-zinc-800/50 hover:bg-zinc-800 disabled:opacity-30 disabled:cursor-not-allowed text-zinc-300 text-[11px]"
                      >
                        Next
                      </button>
                    </div>
                  </div>
                </div>
              </>
            )}
          </div>
        )}

        {/* TAB 2: PRODUCT SALES VIEW */}
        {activeTab === "sales_report" && (
          <div className="space-y-4">
            {salesUploadError && (
              <div className="p-3 bg-red-950/30 border border-red-800/50 rounded-lg flex items-center space-x-2 text-red-300 text-xs">
                <AlertCircle className="w-4 h-4 shrink-0 text-red-400" />
                <span>{salesUploadError}</span>
              </div>
            )}

            {rawSalesData.length === 0 ? (
              <div 
                onClick={() => salesFileInputRef.current?.click()}
                className="cursor-pointer border border-dashed border-zinc-800 hover:border-zinc-700 rounded-xl p-16 text-center bg-[#0d0f17]/40 hover:bg-[#0d0f17]/70 transition flex flex-col items-center justify-center space-y-3"
              >
                <div className="w-10 h-10 rounded-lg bg-zinc-800/80 border border-zinc-700/60 flex items-center justify-center text-zinc-400">
                  <FileText className="w-5 h-5" />
                </div>
                <div className="space-y-1">
                  <p className="text-sm font-medium text-zinc-200">Import Product-Wise Sales Report</p>
                  <p className="text-xs text-zinc-500">Auto-detects Product Types and prompts for selection</p>
                </div>
              </div>
            ) : (
              <>
                <div className="flex flex-col md:flex-row items-stretch md:items-center justify-between gap-3">
                  <div className="flex flex-col sm:flex-row items-stretch sm:items-center gap-2 flex-1 max-w-xl">
                    <div className="relative flex-1">
                      <Search className="w-3.5 h-3.5 absolute left-3 top-2.5 text-zinc-500" />
                      <input
                        type="text"
                        value={salesSearch}
                        onChange={(e) => {
                          setSalesSearch(e.target.value);
                          setSalesPage(1);
                        }}
                        placeholder="Search code or serial..."
                        className="w-full bg-[#0d0f17] border border-zinc-800 rounded-md pl-8 pr-7 py-1.5 text-xs text-zinc-200 placeholder-zinc-500 focus:outline-none focus:border-zinc-600 transition"
                      />
                      {salesSearch && (
                        <button
                          onClick={() => setSalesSearch("")}
                          className="absolute right-2.5 top-2 text-zinc-500 hover:text-zinc-300"
                        >
                          <X className="w-3.5 h-3.5" />
                        </button>
                      )}
                    </div>

                    <button
                      onClick={openProductTypeModal}
                      className="h-8 px-3 rounded-md bg-[#0d0f17] hover:bg-zinc-800/80 border border-zinc-800 text-xs text-zinc-300 inline-flex items-center space-x-1.5 transition active:scale-[0.98]"
                    >
                      <Filter className="w-3.5 h-3.5 text-zinc-400" />
                      <span>Product Types</span>
                      <span className="ml-1 px-1.5 py-0.2 bg-zinc-800 text-zinc-300 rounded text-[10px] font-mono border border-zinc-700/50">
                        {selectedProductTypes.size}/{productTypeStats.length}
                      </span>
                    </button>
                  </div>

                  <div className="flex items-center space-x-2 self-end md:self-auto">
                    <button
                      onClick={copyAllSerials}
                      className="h-8 px-3 rounded-md bg-zinc-900 hover:bg-zinc-800 border border-zinc-800 text-xs text-zinc-300 inline-flex items-center space-x-1.5 transition"
                    >
                      {copiedSerial === "ALL" ? <Check className="w-3.5 h-3.5 text-emerald-400" /> : <Copy className="w-3.5 h-3.5 text-zinc-400" />}
                      <span>{copiedSerial === "ALL" ? "Copied" : "Copy Serials"}</span>
                    </button>

                    <button
                      onClick={exportCleanSalesCSV}
                      className="h-8 px-3 rounded-md bg-zinc-900 hover:bg-zinc-800 border border-zinc-800 text-xs text-zinc-300 inline-flex items-center space-x-1.5 transition"
                    >
                      <Download className="w-3.5 h-3.5 text-zinc-400" />
                      <span>Export CSV</span>
                    </button>
                  </div>
                </div>

                <div className="rounded-lg border border-zinc-800 bg-[#0d0f17] overflow-hidden">
                  <div className="overflow-x-auto">
                    <table className="w-full text-left border-collapse text-xs">
                      <thead>
                        <tr className="border-b border-zinc-800 text-zinc-400 bg-zinc-900/50">
                          <th className="py-2.5 px-3 font-medium text-zinc-500 w-12 text-center">#</th>
                          
                          <th 
                            onClick={() => handleSalesSort("productType")}
                            className="py-2.5 px-3 font-medium cursor-pointer hover:text-zinc-200 transition group select-none whitespace-nowrap"
                          >
                            <span className="inline-flex items-center">
                              Product Type {renderSortIndicator("productType", salesSortField, salesSortOrder)}
                            </span>
                          </th>

                          <th 
                            onClick={() => handleSalesSort("productCode")}
                            className="py-2.5 px-3 font-medium cursor-pointer hover:text-zinc-200 transition group select-none whitespace-nowrap"
                          >
                            <span className="inline-flex items-center">
                              Product Code {renderSortIndicator("productCode", salesSortField, salesSortOrder)}
                            </span>
                          </th>

                          <th 
                            onClick={() => handleSalesSort("serialNumber")}
                            className="py-2.5 px-3 font-medium cursor-pointer hover:text-zinc-200 transition group select-none whitespace-nowrap"
                          >
                            <span className="inline-flex items-center">
                              Serial Number {renderSortIndicator("serialNumber", salesSortField, salesSortOrder)}
                            </span>
                          </th>
                        </tr>
                      </thead>

                      <tbody className="divide-y divide-zinc-800/60 font-mono text-[11px]">
                        {paginatedSalesData.length === 0 ? (
                          <tr>
                            <td colSpan="4" className="py-10 text-center text-zinc-500 font-sans">
                              {selectedProductTypes.size === 0 
                                ? "No Product Types selected. Click 'Product Types' button above to select types."
                                : "No records match your criteria."}
                            </td>
                          </tr>
                        ) : (
                          paginatedSalesData.map((row, idx) => (
                            <tr key={row.id || idx} className="hover:bg-zinc-800/30 transition-colors">
                              <td className="py-2 px-3 text-center text-zinc-600">
                                {(salesPage - 1) * rowsPerPage + idx + 1}
                              </td>

                              <td className="py-2 px-3 text-zinc-200 font-sans whitespace-nowrap">
                                <span className="px-2 py-0.5 rounded bg-zinc-800/60 border border-zinc-700/40 text-[11px]">
                                  {row.productType}
                                </span>
                              </td>

                              <td className="py-2 px-3 text-zinc-400 whitespace-nowrap font-mono">
                                {row.productCode}
                              </td>

                              <td className="py-2 px-3 text-zinc-200 whitespace-nowrap">
                                <div className="flex items-center space-x-2">
                                  <span>{row.serialNumber}</span>
                                  {row.serialNumber && row.serialNumber !== "-" && (
                                    <button
                                      onClick={() => copyToClipboard(row.serialNumber, setCopiedSerial)}
                                      className="p-1 rounded text-zinc-500 hover:text-zinc-300 transition"
                                      title="Copy Serial Number"
                                    >
                                      {copiedSerial === row.serialNumber ? (
                                        <Check className="w-3 h-3 text-emerald-400" />
                                      ) : (
                                        <Copy className="w-3 h-3" />
                                      )}
                                    </button>
                                  )}
                                </div>
                              </td>
                            </tr>
                          ))
                        )}
                      </tbody>
                    </table>
                  </div>

                  <div className="h-10 px-4 border-t border-zinc-800 bg-zinc-900/30 flex items-center justify-between text-xs text-zinc-500">
                    <div>
                      {processedSalesData.length > 0 ? (
                        <span>
                          {(salesPage - 1) * rowsPerPage + 1}–{Math.min(salesPage * rowsPerPage, processedSalesData.length)} of {processedSalesData.length}
                          {" "}(Selected {selectedProductTypes.size} of {productTypeStats.length} types)
                        </span>
                      ) : (
                        <span>0 records</span>
                      )}
                    </div>

                    <div className="flex items-center space-x-1">
                      <button
                        disabled={salesPage === 1}
                        onClick={() => setSalesPage(p => Math.max(1, p - 1))}
                        className="px-2 py-1 rounded bg-zinc-800/50 hover:bg-zinc-800 disabled:opacity-30 disabled:cursor-not-allowed text-zinc-300 text-[11px]"
                      >
                        Prev
                      </button>
                      <span className="px-2 text-zinc-400 text-[11px]">
                        {salesPage} / {salesTotalPages}
                      </span>
                      <button
                        disabled={salesPage >= salesTotalPages}
                        onClick={() => setSalesPage(p => Math.min(salesTotalPages, p + 1))}
                        className="px-2 py-1 rounded bg-zinc-800/50 hover:bg-zinc-800 disabled:opacity-30 disabled:cursor-not-allowed text-zinc-300 text-[11px]"
                      >
                        Next
                      </button>
                    </div>
                  </div>
                </div>
              </>
            )}
          </div>
        )}

        {/* TAB 3: EPOS RECONCILIATION & CROSS-CHECK VIEW */}
        {activeTab === "epos_reconciliation" && (
          <div className="space-y-5">
            {/* Prerequisite Check: If files are missing, show actionable guide */}
            {(!imeiData.length || !rawSalesData.length) && (
              <div className="p-4 rounded-xl bg-zinc-900/40 border border-zinc-800 text-xs flex flex-col sm:flex-row items-start sm:items-center justify-between gap-3">
                <div className="flex items-center space-x-3">
                  <div className="w-8 h-8 rounded-lg bg-amber-500/10 border border-amber-500/30 flex items-center justify-center shrink-0">
                    <AlertCircle className="w-4 h-4 text-amber-400" />
                  </div>
                  <div className="space-y-0.5">
                    <p className="font-medium text-zinc-200">Reconciliation Setup</p>
                    <p className="text-zinc-500">
                      Upload both <span className={imeiData.length ? "text-emerald-400" : "text-amber-400"}>IMEI Activation</span> and <span className={rawSalesData.length ? "text-emerald-400" : "text-amber-400"}>Product Sales</span> files to perform cross-check.
                    </p>
                  </div>
                </div>
                <div className="flex items-center space-x-2 shrink-0">
                  {!imeiData.length && (
                    <button
                      onClick={() => setActiveTab("imei_activation")}
                      className="px-3 py-1.5 rounded-md bg-zinc-800 hover:bg-zinc-700 text-zinc-200 text-xs font-medium inline-flex items-center space-x-1"
                    >
                      <span>Upload IMEI Report</span>
                      <ArrowRight className="w-3 h-3 text-zinc-400" />
                    </button>
                  )}
                  {!rawSalesData.length && (
                    <button
                      onClick={() => setActiveTab("sales_report")}
                      className="px-3 py-1.5 rounded-md bg-zinc-800 hover:bg-zinc-700 text-zinc-200 text-xs font-medium inline-flex items-center space-x-1"
                    >
                      <span>Upload Sales File</span>
                      <ArrowRight className="w-3 h-3 text-zinc-400" />
                    </button>
                  )}
                </div>
              </div>
            )}

            {/* Reconciliation KPI Metrics Header */}
            <div className="grid grid-cols-2 sm:grid-cols-4 gap-3">
              <div className="p-3.5 rounded-lg border border-zinc-800 bg-[#0d0f17]">
                <div className="flex items-center justify-between text-zinc-500 text-xs mb-1">
                  <span>Total POS Sold</span>
                  <ShoppingBag className="w-3.5 h-3.5" />
                </div>
                <div className="text-lg font-semibold text-white font-mono">
                  {reconciliationData.counts.totalSold}
                </div>
              </div>

              <div className="p-3.5 rounded-lg border border-zinc-800 bg-[#0d0f17]">
                <div className="flex items-center justify-between text-zinc-500 text-xs mb-1">
                  <span>Activated (Matched)</span>
                  <CheckCircle2 className="w-3.5 h-3.5 text-emerald-400" />
                </div>
                <div className="text-lg font-semibold text-emerald-400 font-mono flex items-baseline space-x-1.5">
                  <span>{reconciliationData.counts.matched}</span>
                  <span className="text-xs text-zinc-500 font-normal">
                    ({reconciliationData.rate}%)
                  </span>
                </div>
              </div>

              <div className="p-3.5 rounded-lg border border-zinc-800 bg-[#0d0f17]">
                <div className="flex items-center justify-between text-zinc-500 text-xs mb-1">
                  <span>Pending Activation</span>
                  <Clock className="w-3.5 h-3.5 text-amber-400" />
                </div>
                <div className="text-lg font-semibold text-amber-400 font-mono">
                  {reconciliationData.counts.pending}
                </div>
              </div>

              <div className="p-3.5 rounded-lg border border-zinc-800 bg-[#0d0f17]">
                <div className="flex items-center justify-between text-zinc-500 text-xs mb-1">
                  <span>Direct / Not in POS</span>
                  <HelpCircle className="w-3.5 h-3.5 text-indigo-400" />
                </div>
                <div className="text-lg font-semibold text-indigo-300 font-mono">
                  {reconciliationData.counts.notInPos}
                </div>
              </div>
            </div>

            {/* Reconciliation Filters & Action Bar */}
            <div className="flex flex-col md:flex-row items-stretch md:items-center justify-between gap-3">
              <div className="flex flex-col sm:flex-row items-stretch sm:items-center gap-2 flex-1">
                {/* Search Bar */}
                <div className="relative flex-1 max-w-sm">
                  <Search className="w-3.5 h-3.5 absolute left-3 top-2.5 text-zinc-500" />
                  <input
                    type="text"
                    value={reconSearch}
                    onChange={(e) => {
                      setReconSearch(e.target.value);
                      setReconPage(1);
                    }}
                    placeholder="Search Serial, IMEI or Code..."
                    className="w-full bg-[#0d0f17] border border-zinc-800 rounded-md pl-8 pr-7 py-1.5 text-xs text-zinc-200 placeholder-zinc-500 focus:outline-none focus:border-zinc-600 transition"
                  />
                  {reconSearch && (
                    <button
                      onClick={() => setReconSearch("")}
                      className="absolute right-2.5 top-2 text-zinc-500 hover:text-zinc-300"
                    >
                      <X className="w-3.5 h-3.5" />
                    </button>
                  )}
                </div>

                {/* Filter Tabs / Pills */}
                <div className="flex items-center space-x-1 bg-zinc-900/60 p-0.5 rounded-md border border-zinc-800/80 text-[11px] overflow-x-auto">
                  <button
                    onClick={() => { setReconFilter("all"); setReconPage(1); }}
                    className={`px-2.5 py-1 rounded transition ${
                      reconFilter === "all" 
                        ? "bg-zinc-800 text-white font-medium shadow-sm" 
                        : "text-zinc-400 hover:text-zinc-200"
                    }`}
                  >
                    All ({reconciliationData.items.length})
                  </button>
                  <button
                    onClick={() => { setReconFilter("activated"); setReconPage(1); }}
                    className={`px-2.5 py-1 rounded transition ${
                      reconFilter === "activated" 
                        ? "bg-emerald-950/60 text-emerald-300 border border-emerald-800/50 font-medium" 
                        : "text-zinc-400 hover:text-zinc-200"
                    }`}
                  >
                    Activated ({reconciliationData.counts.matched})
                  </button>
                  <button
                    onClick={() => { setReconFilter("pending"); setReconPage(1); }}
                    className={`px-2.5 py-1 rounded transition ${
                      reconFilter === "pending" 
                        ? "bg-amber-950/60 text-amber-300 border border-amber-800/50 font-medium" 
                        : "text-zinc-400 hover:text-zinc-200"
                    }`}
                  >
                    Pending ({reconciliationData.counts.pending})
                  </button>
                  <button
                    onClick={() => { setReconFilter("not_in_pos"); setReconPage(1); }}
                    className={`px-2.5 py-1 rounded transition ${
                      reconFilter === "not_in_pos" 
                        ? "bg-indigo-950/60 text-indigo-300 border border-indigo-800/50 font-medium" 
                        : "text-zinc-400 hover:text-zinc-200"
                    }`}
                  >
                    Not in POS ({reconciliationData.counts.notInPos})
                  </button>
                </div>
              </div>

              {/* Action Buttons */}
              <div className="flex items-center space-x-2 self-end md:self-auto">
                <button
                  onClick={copyPendingSerials}
                  disabled={reconciliationData.counts.pending === 0}
                  className="h-8 px-3 rounded-md bg-zinc-900 hover:bg-zinc-800 disabled:opacity-40 border border-zinc-800 text-xs text-zinc-300 inline-flex items-center space-x-1.5 transition active:scale-[0.98]"
                  title="Copy all pending activation serials"
                >
                  {copiedPending ? <Check className="w-3.5 h-3.5 text-emerald-400" /> : <Copy className="w-3.5 h-3.5 text-zinc-400" />}
                  <span>{copiedPending ? "Copied" : "Copy Pending Serials"}</span>
                </button>
              </div>
            </div>

            {/* Reconciliation Data Table */}
            <div className="rounded-lg border border-zinc-800 bg-[#0d0f17] overflow-hidden">
              <div className="overflow-x-auto">
                <table className="w-full text-left border-collapse text-xs">
                  <thead>
                    <tr className="border-b border-zinc-800 text-zinc-400 bg-zinc-900/50">
                      <th className="py-2.5 px-3 font-medium text-zinc-500 w-12 text-center">#</th>
                      <th className="py-2.5 px-3 font-medium">Status</th>
                      <th className="py-2.5 px-3 font-medium">Serial / IMEI</th>
                      <th className="py-2.5 px-3 font-medium">Product Type</th>
                      <th className="py-2.5 px-3 font-medium">Product Code</th>
                      <th className="py-2.5 px-3 font-medium">Model</th>
                      <th className="py-2.5 px-3 font-medium">Activation Date</th>
                      <th className="py-2.5 px-3 font-medium text-right">MRP</th>
                    </tr>
                  </thead>

                  <tbody className="divide-y divide-zinc-800/60 font-mono text-[11px]">
                    {paginatedReconData.length === 0 ? (
                      <tr>
                        <td colSpan="8" className="py-12 text-center text-zinc-500 font-sans">
                          {reconciliationData.items.length === 0 
                            ? "No data to reconcile. Please upload files in IMEI Activation and Product Sales tabs."
                            : "No records found matching current filter."}
                        </td>
                      </tr>
                    ) : (
                      paginatedReconData.map((row, idx) => (
                        <tr key={row.id || idx} className="hover:bg-zinc-800/30 transition-colors">
                          <td className="py-2 px-3 text-center text-zinc-600">
                            {(reconPage - 1) * rowsPerPage + idx + 1}
                          </td>

                          <td className="py-2 px-3 whitespace-nowrap font-sans">
                            {row.status === "activated" && (
                              <span className="inline-flex items-center space-x-1 px-2 py-0.5 rounded-full text-[10px] font-medium bg-emerald-950/60 text-emerald-400 border border-emerald-800/60">
                                <span className="w-1.5 h-1.5 rounded-full bg-emerald-400"></span>
                                <span>Activated</span>
                              </span>
                            )}
                            {row.status === "pending" && (
                              <span className="inline-flex items-center space-x-1 px-2 py-0.5 rounded-full text-[10px] font-medium bg-amber-950/60 text-amber-300 border border-amber-800/60">
                                <span className="w-1.5 h-1.5 rounded-full bg-amber-400"></span>
                                <span>Pending Activation</span>
                              </span>
                            )}
                            {row.status === "not_in_pos" && (
                              <span className="inline-flex items-center space-x-1 px-2 py-0.5 rounded-full text-[10px] font-medium bg-indigo-950/60 text-indigo-300 border border-indigo-800/60">
                                <span className="w-1.5 h-1.5 rounded-full bg-indigo-400"></span>
                                <span>Not in POS</span>
                              </span>
                            )}
                          </td>

                          <td className="py-2 px-3 text-zinc-200 whitespace-nowrap">
                            <div className="flex items-center space-x-2">
                              <span>{row.identifier}</span>
                              {row.identifier && row.identifier !== "-" && (
                                <button
                                  onClick={() => copyToClipboard(row.identifier, setCopiedSerial)}
                                  className="p-1 rounded text-zinc-500 hover:text-zinc-300 transition"
                                  title="Copy"
                                >
                                  {copiedSerial === row.identifier ? (
                                    <Check className="w-3 h-3 text-emerald-400" />
                                  ) : (
                                    <Copy className="w-3 h-3" />
                                  )}
                                </button>
                              )}
                            </div>
                          </td>

                          <td className="py-2 px-3 text-zinc-400 font-sans whitespace-nowrap">
                            {row.productType}
                          </td>

                          <td className="py-2 px-3 text-zinc-400 whitespace-nowrap">
                            {row.productCode}
                          </td>

                          <td className="py-2 px-3 text-zinc-300 font-sans whitespace-nowrap">
                            {row.modelNo}
                          </td>

                          <td className="py-2 px-3 text-zinc-300 whitespace-nowrap">
                            {row.activationDate}
                          </td>

                          <td className="py-2 px-3 text-right text-zinc-200 whitespace-nowrap">
                            {row.mrp !== "-" && !row.mrp.startsWith("₹") ? `₹${row.mrp}` : row.mrp}
                          </td>
                        </tr>
                      ))
                    )}
                  </tbody>
                </table>
              </div>

              {/* Reconciliation Pagination Footer */}
              <div className="h-10 px-4 border-t border-zinc-800 bg-zinc-900/30 flex items-center justify-between text-xs text-zinc-500">
                <div>
                  {filteredReconData.length > 0 ? (
                    <span>
                      {(reconPage - 1) * rowsPerPage + 1}–{Math.min(reconPage * rowsPerPage, filteredReconData.length)} of {filteredReconData.length} records
                    </span>
                  ) : (
                    <span>0 records</span>
                  )}
                </div>

                <div className="flex items-center space-x-1">
                  <button
                    disabled={reconPage === 1}
                    onClick={() => setReconPage(p => Math.max(1, p - 1))}
                    className="px-2 py-1 rounded bg-zinc-800/50 hover:bg-zinc-800 disabled:opacity-30 disabled:cursor-not-allowed text-zinc-300 text-[11px]"
                  >
                    Prev
                  </button>
                  <span className="px-2 text-zinc-400 text-[11px]">
                    {reconPage} / {reconTotalPages}
                  </span>
                  <button
                    disabled={reconPage >= reconTotalPages}
                    onClick={() => setReconPage(p => Math.min(reconTotalPages, p + 1))}
                    className="px-2 py-1 rounded bg-zinc-800/50 hover:bg-zinc-800 disabled:opacity-30 disabled:cursor-not-allowed text-zinc-300 text-[11px]"
                  >
                    Next
                  </button>
                </div>
              </div>
            </div>
          </div>
        )}
      </main>

      {}
      {/* Product Type Selection Modal */}
      {isTypeModalOpen && (
        <div className="fixed inset-0 z-50 flex items-center justify-center p-4 bg-black/70 backdrop-blur-sm animate-fadeIn">
          <div className="bg-[#0f111a] border border-zinc-700/80 rounded-xl w-full max-w-md shadow-2xl overflow-hidden flex flex-col max-h-[85vh]">
            
            <div className="px-5 py-4 border-b border-zinc-800 flex items-center justify-between bg-zinc-900/60">
              <div>
                <h3 className="text-sm font-semibold text-white">Select Product Types</h3>
                <p className="text-xs text-zinc-400">
                  Choose which types to display in the table
                </p>
              </div>
              <button
                onClick={() => setIsTypeModalOpen(false)}
                className="text-zinc-400 hover:text-zinc-200 p-1 rounded-md hover:bg-zinc-800 transition"
              >
                <X className="w-4 h-4" />
              </button>
            </div>

            <div className="px-5 py-2.5 border-b border-zinc-800/60 bg-zinc-900/30 flex items-center justify-between text-xs">
              <span className="text-zinc-400">
                {modalTempSelected.size} of {productTypeStats.length} selected
              </span>
              <div className="space-x-2">
                <button
                  type="button"
                  onClick={selectAllModalTypes}
                  className="text-xs text-zinc-300 hover:text-white underline underline-offset-2"
                >
                  Select All
                </button>
                <span className="text-zinc-600">|</span>
                <button
                  type="button"
                  onClick={deselectAllModalTypes}
                  className="text-xs text-zinc-400 hover:text-zinc-200 underline underline-offset-2"
                >
                  Deselect All
                </button>
              </div>
            </div>

            <div className="px-5 py-3 overflow-y-auto space-y-1.5 flex-1 divide-y divide-zinc-800/40">
              {productTypeStats.map(({ type, count }) => {
                const isChecked = modalTempSelected.has(type);
                return (
                  <label
                    key={type}
                    onClick={() => toggleModalItem(type)}
                    className="flex items-center justify-between py-2 px-2 rounded-lg hover:bg-zinc-800/40 cursor-pointer select-none transition group"
                  >
                    <div className="flex items-center space-x-2.5">
                      <div className="text-zinc-400 group-hover:text-zinc-200">
                        {isChecked ? (
                          <CheckSquare className="w-4 h-4 text-emerald-400" />
                        ) : (
                          <Square className="w-4 h-4 text-zinc-600" />
                        )}
                      </div>
                      <span className={`text-xs ${isChecked ? 'text-zinc-200 font-medium' : 'text-zinc-400'}`}>
                        {type}
                      </span>
                    </div>
                    <span className="text-[11px] font-mono text-zinc-500 bg-zinc-800/80 px-2 py-0.5 rounded border border-zinc-700/40">
                      {count} {count === 1 ? 'item' : 'items'}
                    </span>
                  </label>
                );
              })}
            </div>

            <div className="px-5 py-3 border-t border-zinc-800 bg-zinc-900/60 flex items-center justify-end space-x-2">
              <button
                type="button"
                onClick={() => setIsTypeModalOpen(false)}
                className="px-3.5 py-1.5 rounded-md text-xs font-medium text-zinc-400 hover:text-zinc-200 hover:bg-zinc-800 transition"
              >
                Cancel
              </button>
              <button
                type="button"
                onClick={applyModalSelection}
                className="px-4 py-1.5 rounded-md text-xs font-medium bg-zinc-100 hover:bg-white text-zinc-900 transition shadow"
              >
                Apply Selection ({modalTempSelected.size})
              </button>
            </div>

          </div>
        </div>
      )}

      {/* Clear Confirmation Modal */}
      {isClearModalOpen && (
        <div className="fixed inset-0 z-50 flex items-center justify-center p-4 bg-black/70 backdrop-blur-sm">
          <div className="bg-[#0f111a] border border-zinc-700/80 rounded-xl w-full max-w-sm shadow-2xl p-5 space-y-4">
            <div className="flex items-start space-x-3">
              <div className="w-9 h-9 rounded-lg bg-red-950/60 border border-red-800/60 flex items-center justify-center shrink-0 mt-0.5">
                <Trash2 className="w-4 h-4 text-red-400" />
              </div>
              <div className="space-y-1">
                <h3 className="text-sm font-semibold text-white">Clear All Records?</h3>
                <p className="text-xs text-zinc-400 leading-relaxed">
                  Are you sure you want to clear the uploaded data for{" "}
                  <span className="text-zinc-200 font-medium">
                    {activeTab === "imei_activation" 
                      ? "IMEI Activation" 
                      : activeTab === "sales_report" 
                      ? "Product Sales" 
                      : "Both Reports"}
                  </span>
                  ? This will reset the table and search filters.
                </p>
              </div>
            </div>

            <div className="flex items-center justify-end space-x-2 pt-2 border-t border-zinc-800/80">
              <button
                type="button"
                onClick={() => setIsClearModalOpen(false)}
                className="px-3 py-1.5 rounded-md text-xs font-medium text-zinc-400 hover:text-zinc-200 hover:bg-zinc-800 transition"
              >
                Cancel
              </button>
              <button
                type="button"
                onClick={handleConfirmClear}
                className="px-3.5 py-1.5 rounded-md text-xs font-medium bg-red-600 hover:bg-red-500 text-white transition shadow"
              >
                Yes, Clear All
              </button>
            </div>
          </div>
        </div>
      )}

      {}
      <nav className="fixed bottom-0 inset-x-0 h-14 bg-[#0d0f17]/95 backdrop-blur border-t border-zinc-800/80 z-40 px-6 flex items-center justify-center">
        <div className="flex items-center space-x-2">
          {/* Tab 1: IMEI Activation */}
          <button
            onClick={() => setActiveTab("imei_activation")}
            className={`flex items-center space-x-2 px-3.5 py-1.5 rounded-md text-xs font-medium transition ${
              activeTab === "imei_activation"
                ? "bg-zinc-800 text-white border border-zinc-700/60 shadow-sm"
                : "text-zinc-500 hover:text-zinc-300"
            }`}
          >
            <Smartphone className="w-3.5 h-3.5" />
            <span>IMEI Activation</span>
            {imeiData.length > 0 && (
              <span className="ml-1 text-[10px] px-1.5 py-0.2 rounded-full bg-zinc-700/70 text-zinc-300">
                {imeiData.length}
              </span>
            )}
          </button>

          {/* Tab 2: Product Sales */}
          <button
            onClick={() => setActiveTab("sales_report")}
            className={`flex items-center space-x-2 px-3.5 py-1.5 rounded-md text-xs font-medium transition ${
              activeTab === "sales_report"
                ? "bg-zinc-800 text-white border border-zinc-700/60 shadow-sm"
                : "text-zinc-500 hover:text-zinc-300"
            }`}
          >
            <ShoppingBag className="w-3.5 h-3.5" />
            <span>Product Sales</span>
            {rawSalesData.length > 0 && (
              <span className="ml-1 text-[10px] px-1.5 py-0.2 rounded-full bg-zinc-700/70 text-zinc-300">
                {rawSalesData.length}
              </span>
            )}
          </button>

          {/* Tab 3: EPOS Reconciliation */}
          <button
            onClick={() => setActiveTab("epos_reconciliation")}
            className={`flex items-center space-x-2 px-3.5 py-1.5 rounded-md text-xs font-medium transition ${
              activeTab === "epos_reconciliation"
                ? "bg-zinc-800 text-white border border-zinc-700/60 shadow-sm"
                : "text-zinc-500 hover:text-zinc-300"
            }`}
          >
            <GitCompare className="w-3.5 h-3.5 text-emerald-400" />
            <span>EPOS Recon</span>
            {(imeiData.length > 0 || rawSalesData.length > 0) && (
              <span className={`ml-1 text-[10px] px-1.5 py-0.2 rounded-full ${
                reconciliationData.counts.pending > 0 
                  ? 'bg-amber-950/70 text-amber-300 border border-amber-800/40' 
                  : 'bg-emerald-950/70 text-emerald-300 border border-emerald-800/40'
              }`}>
                {reconciliationData.rate}%
              </span>
            )}
          </button>
        </div>
      </nav>

    </div>
  );
}