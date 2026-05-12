import React, { useMemo, useState } from "react";

const LOAN_PROGRAMS = {
  conventional: {
    label: "Conventional",
    maxLtv: {
      purchase: { primary: 97, second_home: 90, investment: 85 },
      rateterm: { primary: 97, second_home: 90, investment: 75 },
      cashout: { primary: 80, second_home: 75, investment: 75 },
    },
    notes: "PMI may apply when LTV is above 80%. Pricing can vary by credit score, occupancy, property type, and LLPAs.",
  },
  fha: {
    label: "FHA",
    maxLtv: {
      purchase: { primary: 96.5, second_home: 0, investment: 0 },
      rateterm: { primary: 97.75, second_home: 0, investment: 0 },
      cashout: { primary: 80, second_home: 0, investment: 0 },
    },
    notes: "FHA is generally for primary residences. Upfront and monthly mortgage insurance may apply.",
  },
  va: {
    label: "VA",
    maxLtv: {
      purchase: { primary: 100, second_home: 0, investment: 0 },
      rateterm: { primary: 100, second_home: 0, investment: 0 },
      cashout: { primary: 90, second_home: 0, investment: 0 },
    },
    notes: "VA eligibility, entitlement, residual income, and funding fee must be verified.",
  },
  usda: {
    label: "USDA",
    maxLtv: {
      purchase: { primary: 100, second_home: 0, investment: 0 },
      rateterm: { primary: 100, second_home: 0, investment: 0 },
      cashout: { primary: 0, second_home: 0, investment: 0 },
    },
    notes: "USDA eligibility depends on property location, household income limits, occupancy, and agency guidelines. Cash-out is generally not supported.",
  },
};

const DEFAULT_SCENARIO = {
  clientName: "",
  clientPhone: "",
  clientEmail: "",
  realtorName: "",
  loanOfficerName: "Yulissa Poux",
  companyName: "Level Mortgage",
  nmlsId: "",
  scenarioDate: new Date().toISOString().slice(0, 10),
  language: "english",
  purpose: "purchase",
  loanType: "fha",
  occupancy: "primary",
  propertyType: "single_family",
  homeValue: "300000",
  purchasePrice: "300000",
  firstMortgageBalance: "0",
  helocBalance: "0",
  helocPayoff: "yes",
  desiredCashOut: "0",
  downPayment: "10500",
  downPaymentPercent: "3.5",
  creditScore: "680",
  interestRate: "6.75",
  loanTerm: "30",
  annualTaxes: "4800",
  annualInsurance: "2400",
  monthlyHOA: "0",
  monthlyIncome: "6000",
  otherMonthlyDebt: "450",
  monthlyMortgageInsurance: "0",
  useAutoMortgageInsurance: "yes",
  closingCosts: "9000",
  prepaidEscrows: "3000",
  sellerCredits: "0",
  lenderCredits: "0",
  otherCredits: "0",
  fhaUfmipPercent: "1.75",
  financeFhaUfmip: "yes",
  vaFundingFeePercent: "2.15",
  financeVaFundingFee: "yes",
  usdaGuaranteeFeePercent: "1",
  financeUsdaGuaranteeFee: "yes",
  targetBackEndDti: "45",
  estimatedTaxRatePercent: "1.6",
  estimatedInsuranceRatePercent: "0.8",
};

const money = (value) => {
  const number = Number.isFinite(value) ? value : 0;
  return number.toLocaleString("en-US", {
    style: "currency",
    currency: "USD",
    maximumFractionDigits: 0,
  });
};

const percent = (value) => {
  const number = Number.isFinite(value) ? value : 0;
  return `${number.toFixed(2)}%`;
};

const toNumber = (value) => {
  const number = Number(value);
  return Number.isFinite(number) ? Math.max(number, 0) : 0;
};

const getMaxLtv = (loanType, purpose, occupancy) => {
  return LOAN_PROGRAMS[loanType]?.maxLtv?.[purpose]?.[occupancy] ?? 0;
};

const calculateMonthlyPayment = (loanAmount, annualRate, months) => {
  const monthlyRate = annualRate / 100 / 12;
  if (loanAmount <= 0 || months <= 0) return 0;
  if (monthlyRate <= 0) return loanAmount / months;
  return loanAmount * ((monthlyRate * Math.pow(1 + monthlyRate, months)) / (Math.pow(1 + monthlyRate, months) - 1));
};

const estimateConventionalPmiRate = (ltv, creditScore) => {
  if (ltv <= 80) return 0;
  if (creditScore >= 760) return ltv > 95 ? 0.36 : 0.28;
  if (creditScore >= 720) return ltv > 95 ? 0.52 : 0.38;
  if (creditScore >= 680) return ltv > 95 ? 0.78 : 0.55;
  if (creditScore >= 640) return ltv > 95 ? 1.05 : 0.82;
  return ltv > 95 ? 1.35 : 1.05;
};

const estimateAnnualMiRate = (scenario, baseLoanAmount, ltv, creditScore) => {
  if (scenario.useAutoMortgageInsurance !== "yes") return null;
  if (scenario.loanType === "conventional") return estimateConventionalPmiRate(ltv, creditScore);
  if (scenario.loanType === "fha") return baseLoanAmount <= 726200 && ltv <= 95 ? 0.5 : 0.55;
  if (scenario.loanType === "usda") return 0.35;
  return 0;
};

const getProgramFee = (scenario, baseLoanAmount) => {
  if (scenario.loanType === "fha") return baseLoanAmount * (toNumber(scenario.fhaUfmipPercent) / 100);
  if (scenario.loanType === "va") return baseLoanAmount * (toNumber(scenario.vaFundingFeePercent) / 100);
  if (scenario.loanType === "usda") return baseLoanAmount * (toNumber(scenario.usdaGuaranteeFeePercent) / 100);
  return 0;
};

const isProgramFeeFinanced = (scenario) => {
  if (scenario.loanType === "fha") return scenario.financeFhaUfmip === "yes";
  if (scenario.loanType === "va") return scenario.financeVaFundingFee === "yes";
  if (scenario.loanType === "usda") return scenario.financeUsdaGuaranteeFee === "yes";
  return false;
};

const calculateScenario = (scenario) => {
  const homeValue = toNumber(scenario.homeValue);
  const purchasePrice = toNumber(scenario.purchasePrice);
  const firstMortgageBalance = toNumber(scenario.firstMortgageBalance);
  const helocBalance = toNumber(scenario.helocBalance);
  const desiredCashOut = toNumber(scenario.desiredCashOut);
  const downPayment = toNumber(scenario.downPayment);
  const downPaymentPercent = toNumber(scenario.downPaymentPercent);
  const creditScore = toNumber(scenario.creditScore);
  const interestRate = toNumber(scenario.interestRate);
  const loanTermYears = toNumber(scenario.loanTerm) || 30;
  const months = loanTermYears * 12;
  const taxes = toNumber(scenario.annualTaxes) / 12;
  const insurance = toNumber(scenario.annualInsurance) / 12;
  const hoa = toNumber(scenario.monthlyHOA);
  const income = toNumber(scenario.monthlyIncome);
  const debts = toNumber(scenario.otherMonthlyDebt);
  const manualMortgageInsurance = toNumber(scenario.monthlyMortgageInsurance);
  const closingCosts = toNumber(scenario.closingCosts);
  const prepaidEscrows = toNumber(scenario.prepaidEscrows);
  const sellerCredits = toNumber(scenario.sellerCredits);
  const lenderCredits = toNumber(scenario.lenderCredits);
  const otherCredits = toNumber(scenario.otherCredits);
  const payoffHELOC = scenario.helocPayoff === "yes";
  const isPurchase = scenario.purpose === "purchase";
  const valueBasis = isPurchase ? purchasePrice || homeValue : homeValue;
  const existingLiens = firstMortgageBalance + helocBalance;
  const baseLoanAmount = isPurchase ? Math.max(0, valueBasis - downPayment) : firstMortgageBalance + (payoffHELOC ? helocBalance : 0) + desiredCashOut;
  const ltv = valueBasis > 0 ? (baseLoanAmount / valueBasis) * 100 : 0;
  const annualMiRate = estimateAnnualMiRate(scenario, baseLoanAmount, ltv, creditScore);
  const monthlyMortgageInsurance = annualMiRate === null ? manualMortgageInsurance : (baseLoanAmount * (annualMiRate / 100)) / 12;
  const programFeeAmount = getProgramFee(scenario, baseLoanAmount);
  const financedProgramFee = isProgramFeeFinanced(scenario) ? programFeeAmount : 0;
  const cashProgramFee = isProgramFeeFinanced(scenario) ? 0 : programFeeAmount;
  const proposedLoanAmount = baseLoanAmount + financedProgramFee;
  const totalLtvWithFinancedFee = valueBasis > 0 ? (proposedLoanAmount / valueBasis) * 100 : 0;
  const cltv = valueBasis > 0 ? ((proposedLoanAmount + (isPurchase || payoffHELOC ? 0 : helocBalance)) / valueBasis) * 100 : 0;
  const maxLtv = getMaxLtv(scenario.loanType, scenario.purpose, scenario.occupancy);
  const maxLoanByLTV = valueBasis * (maxLtv / 100);
  const maxCashOutAvailable = isPurchase ? 0 : Math.max(0, maxLoanByLTV - firstMortgageBalance - (payoffHELOC ? helocBalance : 0));
  const estimatedEquity = Math.max(0, homeValue - existingLiens);
  const principalAndInterest = calculateMonthlyPayment(proposedLoanAmount, interestRate, months);
  const totalHousingPayment = principalAndInterest + taxes + insurance + hoa + monthlyMortgageInsurance;
  const frontEndDti = income > 0 ? (totalHousingPayment / income) * 100 : 0;
  const backEndDti = income > 0 ? ((totalHousingPayment + debts) / income) * 100 : 0;
  const cashToClose = isPurchase ? Math.max(0, downPayment + closingCosts + prepaidEscrows + cashProgramFee - sellerCredits - lenderCredits - otherCredits) : Math.max(0, closingCosts + prepaidEscrows + cashProgramFee - lenderCredits - otherCredits);

  return {
    homeValue,
    purchasePrice,
    valueBasis,
    firstMortgageBalance,
    helocBalance,
    desiredCashOut,
    downPayment,
    downPaymentPercent,
    existingLiens,
    baseLoanAmount,
    proposedLoanAmount,
    programFeeAmount,
    financedProgramFee,
    cashProgramFee,
    fhaUfmipAmount: scenario.loanType === "fha" ? programFeeAmount : 0,
    financedFhaUfmip: scenario.loanType === "fha" ? financedProgramFee : 0,
    cashFhaUfmip: scenario.loanType === "fha" ? cashProgramFee : 0,
    vaFundingFeeAmount: scenario.loanType === "va" ? programFeeAmount : 0,
    usdaGuaranteeFeeAmount: scenario.loanType === "usda" ? programFeeAmount : 0,
    annualMiRate: annualMiRate ?? 0,
    ltv,
    totalLtvWithFinancedUfmip: totalLtvWithFinancedFee,
    totalLtvWithFinancedFee,
    cltv,
    maxLtv,
    maxLoanByLTV,
    maxCashOutAvailable,
    estimatedEquity,
    principalAndInterest,
    taxes,
    insurance,
    hoa,
    monthlyMortgageInsurance,
    totalHousingPayment,
    frontEndDti,
    backEndDti,
    dti: backEndDti,
    closingCosts,
    prepaidEscrows,
    sellerCredits,
    lenderCredits,
    otherCredits,
    cashToClose,
  };
};

const calculateMaxPurchase = (scenario) => {
  const income = toNumber(scenario.monthlyIncome);
  const debts = toNumber(scenario.otherMonthlyDebt);
  const targetDti = toNumber(scenario.targetBackEndDti) || 45;
  const maxTotalPayment = Math.max(0, income * (targetDti / 100) - debts);
  const downPaymentPercent = toNumber(scenario.downPaymentPercent);
  const annualTaxRate = toNumber(scenario.estimatedTaxRatePercent) / 100;
  const annualInsuranceRate = toNumber(scenario.estimatedInsuranceRatePercent) / 100;
  const hoa = toNumber(scenario.monthlyHOA);
  const monthlyRate = toNumber(scenario.interestRate) / 100 / 12;
  const months = (toNumber(scenario.loanTerm) || 30) * 12;
  const loanRatio = Math.max(0, 1 - downPaymentPercent / 100);
  const miRate = scenario.loanType === "fha" ? 0.0055 : scenario.loanType === "usda" ? 0.0035 : scenario.loanType === "conventional" ? 0.0055 : 0;
  const paymentFactor = monthlyRate > 0 ? (monthlyRate * Math.pow(1 + monthlyRate, months)) / (Math.pow(1 + monthlyRate, months) - 1) : 1 / months;
  const monthlyCostPerDollar = loanRatio * paymentFactor + annualTaxRate / 12 + annualInsuranceRate / 12 + loanRatio * miRate / 12;
  const maxPurchasePrice = monthlyCostPerDollar > 0 ? Math.max(0, (maxTotalPayment - hoa) / monthlyCostPerDollar) : 0;
  const estimatedDownPayment = maxPurchasePrice * (downPaymentPercent / 100);
  const estimatedLoanAmount = Math.max(0, maxPurchasePrice - estimatedDownPayment);

  return { maxTotalPayment, maxPurchasePrice, estimatedDownPayment, estimatedLoanAmount };
};

const getDocumentChecklist = (scenario) => {
  const base = ["Government-issued photo ID", "Most recent 30 days of paystubs", "Last 2 months of bank statements", "Last 2 years W-2s or 1099s", "Authorization to pull credit", "Homeowners insurance contact or estimate"];
  if (scenario.purpose === "purchase") base.push("Fully executed purchase contract once available", "Earnest money deposit proof");
  if (scenario.purpose !== "purchase") base.push("Current mortgage statement", "Homeowners insurance declarations page", "Most recent property tax bill");
  if (toNumber(scenario.helocBalance) > 0) base.push("Current HELOC statement", "HELOC subordination details if not paying it off");
  if (scenario.loanType === "fha") base.push("FHA case assignment items as required", "Explanation for any large deposits");
  if (scenario.loanType === "va") base.push("Certificate of Eligibility", "DD-214 or statement of service", "VA funding fee exemption documentation if applicable");
  if (scenario.loanType === "usda") base.push("Household income documentation", "Property eligibility confirmation", "USDA income eligibility confirmation");
  return base;
};

const comparePrograms = (scenario) => {
  return ["conventional", "fha", "va", "usda"].map((loanType) => {
    const adjusted = { ...scenario, loanType };
    const result = calculateScenario(adjusted);
    return { loanType, label: LOAN_PROGRAMS[loanType].label, result, eligible: result.maxLtv > 0 && result.ltv <= result.maxLtv };
  });
};

const almostEqual = (actual, expected, tolerance = 0.01) => Math.abs(actual - expected) <= tolerance;

const runCalculationTests = () => {
  const fhaPurchaseScenario = calculateScenario({ ...DEFAULT_SCENARIO, purchasePrice: "300000", downPayment: "10500", homeValue: "300000", fhaUfmipPercent: "1.75", financeFhaUfmip: "yes" });
  const fhaCashUfmipScenario = calculateScenario({ ...DEFAULT_SCENARIO, purchasePrice: "300000", downPayment: "10500", homeValue: "300000", fhaUfmipPercent: "1.75", financeFhaUfmip: "no" });
  const conventionalRefiScenario = calculateScenario({ ...DEFAULT_SCENARIO, purpose: "cashout", loanType: "conventional", homeValue: "298000", purchasePrice: "", firstMortgageBalance: "104000", helocBalance: "118000", helocPayoff: "yes", desiredCashOut: "25000", useAutoMortgageInsurance: "no", monthlyMortgageInsurance: "0" });
  const maxPurchase = calculateMaxPurchase(DEFAULT_SCENARIO);

  return [
    { name: "FHA base loan uses purchase price minus down payment", pass: almostEqual(fhaPurchaseScenario.baseLoanAmount, 289500) && almostEqual(fhaPurchaseScenario.ltv, 96.5) },
    { name: "FHA UFMIP can be financed into the total loan amount", pass: almostEqual(fhaPurchaseScenario.fhaUfmipAmount, 5066.25) && almostEqual(fhaPurchaseScenario.proposedLoanAmount, 294566.25) },
    { name: "FHA UFMIP can be paid in cash and included in cash to close", pass: almostEqual(fhaCashUfmipScenario.cashFhaUfmip, 5066.25) && fhaCashUfmipScenario.cashToClose > fhaPurchaseScenario.cashToClose },
    { name: "Cash-out refinance includes HELOC payoff and cash out", pass: almostEqual(conventionalRefiScenario.proposedLoanAmount, 247000) },
    { name: "Back-end DTI includes full housing payment plus monthly debts", pass: almostEqual(fhaPurchaseScenario.backEndDti, ((fhaPurchaseScenario.totalHousingPayment + 450) / 6000) * 100) },
    { name: "Max purchase calculator returns a usable purchase price", pass: maxPurchase.maxPurchasePrice > 0 && maxPurchase.estimatedLoanAmount > 0 },
  ];
};

export default function LoanPricingAssistant() {
  const [scenario, setScenario] = useState(DEFAULT_SCENARIO);
  const [copyStatus, setCopyStatus] = useState("");

  const numbers = useMemo(() => calculateScenario(scenario), [scenario]);
  const maxPurchase = useMemo(() => calculateMaxPurchase(scenario), [scenario]);
  const programComparison = useMemo(() => comparePrograms(scenario), [scenario]);
  const checklist = useMemo(() => getDocumentChecklist(scenario), [scenario]);
  const tests = useMemo(() => runCalculationTests(), []);
  const testsPassed = tests.every((test) => test.pass);
  const selectedProgram = LOAN_PROGRAMS[scenario.loanType];

  const update = (field, value) => setScenario((prev) => ({ ...prev, [field]: value }));

  const applyMaxPurchase = () => {
    const price = maxPurchase.maxPurchasePrice.toFixed(0);
    setScenario((prev) => ({
      ...prev,
      purpose: "purchase",
      purchasePrice: price,
      homeValue: price,
      downPayment: maxPurchase.estimatedDownPayment.toFixed(2),
    }));
  };

  const updateDownPaymentPercent = (value) => {
    const purchasePrice = toNumber(scenario.purchasePrice || scenario.homeValue);
    const percentage = toNumber(value);
    const downPayment = purchasePrice > 0 ? (purchasePrice * percentage) / 100 : 0;
    setScenario((prev) => ({ ...prev, downPaymentPercent: value, downPayment: downPayment ? downPayment.toFixed(2) : "0" }));
  };

  const updateDownPaymentAmount = (value) => {
    const purchasePrice = toNumber(scenario.purchasePrice || scenario.homeValue);
    const downPayment = toNumber(value);
    const percentage = purchasePrice > 0 ? (downPayment / purchasePrice) * 100 : 0;
    setScenario((prev) => ({ ...prev, downPayment: value, downPaymentPercent: percentage ? percentage.toFixed(3) : "0" }));
  };

  const alerts = useMemo(() => {
    const items = [];
    if (!scenario.clientName) items.push("Client name is missing. Add client information before saving or sending the summary.");
    if (numbers.maxLtv === 0) items.push("This loan purpose, occupancy, and program combination is not supported in this estimate. Confirm current investor guidelines.");
    if (numbers.maxLtv > 0 && numbers.ltv > numbers.maxLtv) items.push(`Base LTV is above the estimated ${numbers.maxLtv}% max for this scenario.`);
    if (numbers.backEndDti > 50) items.push("Estimated back-end DTI is above 50%. Review program limits, compensating factors, AUS findings, and lender overlays.");
    if (numbers.helocBalance > 0 && scenario.helocPayoff === "no") items.push("HELOC is being subordinated, so CLTV/HCLTV matters. Confirm lender limits and subordination approval.");
    if (scenario.purpose === "cashout") items.push("For cash-out, verify seasoning, title ownership, occupancy, and whether any second lien was recently opened or drawn.");
    if (scenario.loanType === "fha") items.push("FHA pricing should verify UFMIP, monthly MIP, FHA case assignment, county loan limits, and lender overlays.");
    if (scenario.loanType === "va") items.push("VA pricing should verify COE, entitlement, funding fee, residual income, and lender overlays.");
    if (scenario.loanType === "usda") items.push("USDA pricing should verify property eligibility, household income limits, guarantee fee, annual fee, and agency overlays.");
    if (!scenario.creditScore) items.push("Credit score is missing. Pricing and eligibility can change significantly by score bucket.");
    return items;
  }, [numbers, scenario.clientName, scenario.creditScore, scenario.helocPayoff, scenario.loanType, scenario.purpose]);

  const complianceDisclaimer = scenario.language === "spanish"
    ? "Este es un estimado preliminar solo para fines informativos y no representa una aprobación de préstamo, compromiso de prestar, bloqueo de tasa ni garantía de términos. La elegibilidad final, tasa, pago, costos, APR, efectivo para cerrar y monto del préstamo están sujetos a solicitud completa, revisión de crédito, verificación de ingresos y activos, tasación, título, ocupación, elegibilidad de la propiedad, aprobación de underwriting/AUS, guías del inversionista y requisitos federales/estatales aplicables. Las tasas, costos, guías y disponibilidad de programas pueden cambiar sin previo aviso. Igualdad de Oportunidad de Vivienda."
    : "This is a preliminary estimate for discussion purposes only and is not a loan approval, commitment to lend, rate lock, or guarantee of terms. Final eligibility, pricing, payment, fees, APR, cash to close, and loan amount are subject to a complete application, credit review, income and asset verification, appraisal, title, occupancy, property eligibility, AUS/underwriting approval, investor guidelines, and applicable federal/state requirements. Rates, fees, guidelines, and program availability are subject to change without notice. This estimate may not include all finance charges, mortgage insurance, government funding/guarantee fees, discount points, escrow adjustments, prepaid items, closing costs, or lender overlays. Equal Housing Opportunity.";

  const clientSummary = useMemo(() => {
    const heading = scenario.language === "spanish" ? "Resumen Preliminar del Préstamo" : "Loan Scenario Summary";
    const clientLabel = scenario.language === "spanish" ? "Cliente" : "Client";
    const paymentLabel = scenario.language === "spanish" ? "Pago mensual estimado" : "Estimated total housing payment";
    const cashLabel = scenario.language === "spanish" ? "Efectivo estimado para cerrar" : "Estimated cash to close";
    return `${heading}\n\n${clientLabel}: ${scenario.clientName || "Not provided"}\nPhone: ${scenario.clientPhone || "Not provided"}\nEmail: ${scenario.clientEmail || "Not provided"}\nLoan Officer: ${scenario.loanOfficerName || "Not provided"}\nCompany: ${scenario.companyName || "Not provided"}\nNMLS: ${scenario.nmlsId || "Not provided"}\nScenario Date: ${scenario.scenarioDate || "Not provided"}\n\nProgram: ${selectedProgram?.label || "Not provided"}\nPurpose: ${scenario.purpose}\nOccupancy: ${scenario.occupancy}\nProperty Type: ${scenario.propertyType}\n\nPurchase price / value basis: ${money(numbers.valueBasis)}\nDown payment: ${money(numbers.downPayment)} (${percent(numbers.downPaymentPercent)})\nBase loan amount: ${money(numbers.baseLoanAmount)}\nProgram fee: ${money(numbers.programFeeAmount)}\nEstimated total loan amount: ${money(numbers.proposedLoanAmount)}\nEstimated base LTV: ${percent(numbers.ltv)}\nEstimated CLTV/HCLTV: ${percent(numbers.cltv)}\nEstimated max LTV used: ${percent(numbers.maxLtv)}\nEstimated P&I: ${money(numbers.principalAndInterest)}\nEstimated monthly mortgage insurance: ${money(numbers.monthlyMortgageInsurance)}\n${paymentLabel}: ${money(numbers.totalHousingPayment)}\nEstimated front-end DTI: ${percent(numbers.frontEndDti)}\nEstimated back-end DTI: ${percent(numbers.backEndDti)}\n${cashLabel}: ${money(numbers.cashToClose)}\nEstimated max purchase power: ${money(maxPurchase.maxPurchasePrice)}\n\nProgram Notes: ${selectedProgram?.notes || "Confirm current program guidelines."}\n\nImportant Disclaimer: ${complianceDisclaimer}`;
  }, [numbers, maxPurchase, scenario, selectedProgram, complianceDisclaimer]);

  const copySummary = async () => {
    if (!window.navigator?.clipboard?.writeText) {
      setCopyStatus("Copy is not available in this browser.");
      return;
    }
    try {
      await window.navigator.clipboard.writeText(clientSummary);
      setCopyStatus("Summary copied successfully.");
      window.setTimeout(() => setCopyStatus(""), 3000);
    } catch {
      setCopyStatus("Unable to copy summary.");
    }
  };

  const printEstimate = () => window.print();

  return (
    <main className="min-h-screen bg-slate-50 p-4 text-slate-900 md:p-8">
      <style>{`
        @media print {
          body * { visibility: hidden; }
          #client-estimate, #client-estimate * { visibility: visible; }
          #client-estimate { position: absolute; left: 0; top: 0; width: 100%; background: white; padding: 24px; }
          .no-print { display: none !important; }
        }
      `}</style>
      <div className="mx-auto max-w-7xl space-y-6">
        <section className="rounded-3xl border bg-white p-6 shadow-sm">
          <div className="flex flex-col gap-3 md:flex-row md:items-center md:justify-between">
            <div>
              <h1 className="text-3xl font-bold tracking-tight">Loan Pricing Assistant</h1>
              <p className="mt-2 text-slate-600">Estimate Conventional, FHA, VA, and USDA scenarios with MI/fees, max purchase power, comparisons, client estimate output, and compliance language.</p>
            </div>
            <div className="rounded-2xl bg-slate-100 px-4 py-3 text-sm text-slate-700">LO Scenario Tool • Preliminary Estimate</div>
          </div>
        </section>

        <section className="rounded-3xl border bg-white p-6 shadow-sm">
          <h2 className="mb-5 text-xl font-semibold">Client & Branding Information</h2>
          <div className="grid grid-cols-1 gap-4 md:grid-cols-3">
            <TextField label="Client Name" value={scenario.clientName} onChange={(value) => update("clientName", value)} />
            <TextField label="Client Phone" value={scenario.clientPhone} onChange={(value) => update("clientPhone", value)} />
            <TextField label="Client Email" value={scenario.clientEmail} onChange={(value) => update("clientEmail", value)} />
            <TextField label="Realtor / Referral Partner" value={scenario.realtorName} onChange={(value) => update("realtorName", value)} />
            <TextField label="Loan Officer Name" value={scenario.loanOfficerName} onChange={(value) => update("loanOfficerName", value)} />
            <TextField label="Company" value={scenario.companyName} onChange={(value) => update("companyName", value)} />
            <TextField label="NMLS ID" value={scenario.nmlsId} onChange={(value) => update("nmlsId", value)} />
            <TextField label="Scenario Date" type="date" value={scenario.scenarioDate} onChange={(value) => update("scenarioDate", value)} />
            <SelectField label="Estimate Language" value={scenario.language} onChange={(value) => update("language", value)} options={[{ value: "english", label: "English" }, { value: "spanish", label: "Spanish" }]} />
          </div>
        </section>

        <section className="grid grid-cols-1 gap-6 lg:grid-cols-3">
          <div className="rounded-3xl border bg-white p-6 shadow-sm lg:col-span-2">
            <h2 className="mb-5 text-xl font-semibold">Loan Scenario Inputs</h2>
            <div className="grid grid-cols-1 gap-4 md:grid-cols-3">
              <SelectField label="Purpose" value={scenario.purpose} onChange={(value) => update("purpose", value)} options={[{ value: "purchase", label: "Purchase" }, { value: "rateterm", label: "Rate/Term Refi" }, { value: "cashout", label: "Cash-Out Refi" }]} />
              <SelectField label="Loan Program" value={scenario.loanType} onChange={(value) => update("loanType", value)} options={[{ value: "conventional", label: "Conventional" }, { value: "fha", label: "FHA" }, { value: "va", label: "VA" }, { value: "usda", label: "USDA" }]} />
              <SelectField label="Occupancy" value={scenario.occupancy} onChange={(value) => update("occupancy", value)} options={[{ value: "primary", label: "Primary" }, { value: "second_home", label: "Second Home" }, { value: "investment", label: "Investment" }]} />
              <SelectField label="Property Type" value={scenario.propertyType} onChange={(value) => update("propertyType", value)} options={[{ value: "single_family", label: "Single Family" }, { value: "condo", label: "Condo" }, { value: "townhome", label: "Townhome" }, { value: "multi_unit", label: "2-4 Unit" }, { value: "manufactured", label: "Manufactured Home" }]} />
              <NumberField label="Home Value / Appraised Value" value={scenario.homeValue} onChange={(value) => update("homeValue", value)} />
              <NumberField label="Purchase Price" value={scenario.purchasePrice} onChange={(value) => update("purchasePrice", value)} />
              <NumberField label="Down Payment %" value={scenario.downPaymentPercent} onChange={updateDownPaymentPercent} />
              <NumberField label="Down Payment $" value={scenario.downPayment} onChange={updateDownPaymentAmount} />
              <NumberField label="Current First Mortgage Balance" value={scenario.firstMortgageBalance} onChange={(value) => update("firstMortgageBalance", value)} />
              <NumberField label="HELOC / Second Lien Balance" value={scenario.helocBalance} onChange={(value) => update("helocBalance", value)} />
              <SelectField label="Pay Off HELOC?" value={scenario.helocPayoff} onChange={(value) => update("helocPayoff", value)} options={[{ value: "yes", label: "Yes, include in new loan" }, { value: "no", label: "No, subordinate it" }]} />
              <NumberField label="Desired Cash Out" value={scenario.desiredCashOut} onChange={(value) => update("desiredCashOut", value)} />
              <NumberField label="Credit Score" value={scenario.creditScore} onChange={(value) => update("creditScore", value)} />
              <NumberField label="Interest Rate %" value={scenario.interestRate} onChange={(value) => update("interestRate", value)} />
              <NumberField label="Loan Term Years" value={scenario.loanTerm} onChange={(value) => update("loanTerm", value)} />
              <NumberField label="Annual Taxes" value={scenario.annualTaxes} onChange={(value) => update("annualTaxes", value)} />
              <NumberField label="Annual Insurance" value={scenario.annualInsurance} onChange={(value) => update("annualInsurance", value)} />
              <NumberField label="Monthly HOA" value={scenario.monthlyHOA} onChange={(value) => update("monthlyHOA", value)} />
              <SelectField label="Auto MI / Fee Estimate?" value={scenario.useAutoMortgageInsurance} onChange={(value) => update("useAutoMortgageInsurance", value)} options={[{ value: "yes", label: "Yes, estimate automatically" }, { value: "no", label: "No, enter manually" }]} />
              <NumberField label="Manual Monthly MI" value={scenario.monthlyMortgageInsurance} onChange={(value) => update("monthlyMortgageInsurance", value)} />
              <NumberField label="Monthly Gross Income" value={scenario.monthlyIncome} onChange={(value) => update("monthlyIncome", value)} />
              <NumberField label="Other Monthly Debts" value={scenario.otherMonthlyDebt} onChange={(value) => update("otherMonthlyDebt", value)} />
            </div>
          </div>
          <div className="space-y-6">
            <MetricCard title="Program" value={selectedProgram?.label || "N/A"} subtitle={selectedProgram?.notes || "Confirm guidelines."} />
            <MetricCard title="Total Loan Amount" value={money(numbers.proposedLoanAmount)} subtitle="Includes financed FHA/VA/USDA fee when applicable" />
            <MetricCard title="Estimated Payment" value={money(numbers.totalHousingPayment)} subtitle="P&I + taxes + insurance + HOA + MI" />
            <MetricCard title="Back-End DTI" value={percent(numbers.backEndDti)} subtitle="Housing payment plus other monthly debts" />
            <MetricCard title="Estimated Cash to Close" value={money(numbers.cashToClose)} subtitle="Down payment + costs/prepaids - credits + cash-paid fees" />
          </div>
        </section>

        <section className="rounded-3xl border bg-white p-6 shadow-sm">
          <h2 className="mb-5 text-xl font-semibold">Program Fees & Mortgage Insurance</h2>
          <div className="grid grid-cols-1 gap-4 md:grid-cols-3">
            <NumberField label="FHA UFMIP %" value={scenario.fhaUfmipPercent} onChange={(value) => update("fhaUfmipPercent", value)} />
            <SelectField label="Finance FHA UFMIP?" value={scenario.financeFhaUfmip} onChange={(value) => update("financeFhaUfmip", value)} options={[{ value: "yes", label: "Yes" }, { value: "no", label: "No" }]} />
            <NumberField label="VA Funding Fee %" value={scenario.vaFundingFeePercent} onChange={(value) => update("vaFundingFeePercent", value)} />
            <SelectField label="Finance VA Fee?" value={scenario.financeVaFundingFee} onChange={(value) => update("financeVaFundingFee", value)} options={[{ value: "yes", label: "Yes" }, { value: "no", label: "No" }]} />
            <NumberField label="USDA Guarantee Fee %" value={scenario.usdaGuaranteeFeePercent} onChange={(value) => update("usdaGuaranteeFeePercent", value)} />
            <SelectField label="Finance USDA Fee?" value={scenario.financeUsdaGuaranteeFee} onChange={(value) => update("financeUsdaGuaranteeFee", value)} options={[{ value: "yes", label: "Yes" }, { value: "no", label: "No" }]} />
            <MetricInline label="Program Fee" value={money(numbers.programFeeAmount)} />
            <MetricInline label="Financed Fee" value={money(numbers.financedProgramFee)} />
            <MetricInline label="Monthly MI Estimate" value={money(numbers.monthlyMortgageInsurance)} />
          </div>
        </section>

        <section className="rounded-3xl border bg-white p-6 shadow-sm">
          <h2 className="mb-5 text-xl font-semibold">Cash to Close & Max Purchase Power</h2>
          <div className="grid grid-cols-1 gap-4 md:grid-cols-3">
            <NumberField label="Estimated Closing Costs" value={scenario.closingCosts} onChange={(value) => update("closingCosts", value)} />
            <NumberField label="Prepaids / Escrows" value={scenario.prepaidEscrows} onChange={(value) => update("prepaidEscrows", value)} />
            <NumberField label="Seller Credits" value={scenario.sellerCredits} onChange={(value) => update("sellerCredits", value)} />
            <NumberField label="Lender Credits" value={scenario.lenderCredits} onChange={(value) => update("lenderCredits", value)} />
            <NumberField label="Other Credits" value={scenario.otherCredits} onChange={(value) => update("otherCredits", value)} />
            <NumberField label="Target Back-End DTI %" value={scenario.targetBackEndDti} onChange={(value) => update("targetBackEndDti", value)} />
            <NumberField label="Estimated Tax Rate %" value={scenario.estimatedTaxRatePercent} onChange={(value) => update("estimatedTaxRatePercent", value)} />
            <NumberField label="Estimated Insurance Rate %" value={scenario.estimatedInsuranceRatePercent} onChange={(value) => update("estimatedInsuranceRatePercent", value)} />
            <MetricInline label="Estimated Cash to Close" value={money(numbers.cashToClose)} />
            <MetricInline label="Max Purchase Price" value={money(maxPurchase.maxPurchasePrice)} />
            <MetricInline label="Max Payment Target" value={money(maxPurchase.maxTotalPayment)} />
            <button className="rounded-2xl bg-slate-900 px-4 py-3 text-sm font-semibold text-white hover:bg-slate-700" type="button" onClick={applyMaxPurchase}>Apply Max Purchase</button>
          </div>
        </section>

        <section className="rounded-3xl border bg-white p-6 shadow-sm">
          <h2 className="mb-5 text-xl font-semibold">Program Comparison</h2>
          <div className="grid grid-cols-1 gap-4 md:grid-cols-4">
            {programComparison.map((item) => (
              <div key={item.loanType} className={`rounded-2xl border p-4 ${item.eligible ? "bg-emerald-50 border-emerald-100" : "bg-amber-50 border-amber-100"}`}>
                <p className="text-sm font-semibold text-slate-900">{item.label}</p>
                <p className="mt-2 text-sm text-slate-600">Payment: {money(item.result.totalHousingPayment)}</p>
                <p className="text-sm text-slate-600">Cash to Close: {money(item.result.cashToClose)}</p>
                <p className="text-sm text-slate-600">DTI: {percent(item.result.backEndDti)}</p>
                <p className="text-sm text-slate-600">LTV: {percent(item.result.ltv)} / Max {percent(item.result.maxLtv)}</p>
                <p className="mt-2 text-xs font-semibold uppercase tracking-wide text-slate-500">{item.eligible ? "Estimate OK" : "Review Needed"}</p>
              </div>
            ))}
          </div>
        </section>

        <section className="rounded-3xl border bg-white p-6 shadow-sm">
          <h2 className="mb-5 text-xl font-semibold">Document Checklist</h2>
          <div className="grid grid-cols-1 gap-3 md:grid-cols-2">
            {checklist.map((item) => (
              <div key={item} className="rounded-2xl border bg-slate-50 p-3 text-sm text-slate-700">□ {item}</div>
            ))}
          </div>
        </section>

        <section id="client-estimate" className="rounded-3xl border bg-white p-6 shadow-sm">
          <div className="mb-6 flex flex-col gap-3 border-b pb-5 md:flex-row md:items-start md:justify-between">
            <div>
              <p className="text-sm font-semibold uppercase tracking-wide text-slate-500">{scenario.language === "spanish" ? "Estimado Preliminar de Préstamo" : "Preliminary Loan Estimate"}</p>
              <h2 className="mt-1 text-3xl font-bold text-slate-900">{scenario.clientName || "Client Name"}</h2>
              <p className="mt-1 text-sm text-slate-600">Prepared on {scenario.scenarioDate || "N/A"}</p>
              <p className="mt-1 text-sm text-slate-600">{scenario.loanOfficerName} • {scenario.companyName} {scenario.nmlsId ? `• NMLS ${scenario.nmlsId}` : ""}</p>
            </div>
            <div className="text-sm text-slate-600 md:text-right">
              <p>{scenario.clientPhone || "Phone not provided"}</p>
              <p>{scenario.clientEmail || "Email not provided"}</p>
              <p>{scenario.realtorName ? `Realtor / Partner: ${scenario.realtorName}` : ""}</p>
            </div>
          </div>
          <div className="grid grid-cols-1 gap-4 md:grid-cols-4">
            <EstimateBox label="Program" value={selectedProgram?.label || "N/A"} />
            <EstimateBox label="Purpose" value={scenario.purpose} />
            <EstimateBox label="Estimated Payment" value={money(numbers.totalHousingPayment)} />
            <EstimateBox label="Cash to Close" value={money(numbers.cashToClose)} />
          </div>
          <div className="mt-6 grid grid-cols-1 gap-6 md:grid-cols-2">
            <div className="rounded-2xl border p-5">
              <h3 className="mb-3 text-lg font-semibold">Loan Details</h3>
              <Line label="Purchase Price / Value Basis" value={money(numbers.valueBasis)} />
              <Line label="Down Payment" value={`${money(numbers.downPayment)} (${percent(numbers.downPaymentPercent)})`} />
              <Line label="Base Loan Amount" value={money(numbers.baseLoanAmount)} />
              <Line label="Program Fee" value={money(numbers.programFeeAmount)} />
              <Line label="Total Loan Amount" value={money(numbers.proposedLoanAmount)} strong />
              <Line label="Base LTV" value={percent(numbers.ltv)} />
              <Line label="CLTV / HCLTV" value={percent(numbers.cltv)} />
            </div>
            <div className="rounded-2xl border p-5">
              <h3 className="mb-3 text-lg font-semibold">Payment & DTI</h3>
              <Line label="Principal & Interest" value={money(numbers.principalAndInterest)} />
              <Line label="Taxes" value={money(numbers.taxes)} />
              <Line label="Insurance" value={money(numbers.insurance)} />
              <Line label="HOA" value={money(numbers.hoa)} />
              <Line label="Mortgage Insurance" value={money(numbers.monthlyMortgageInsurance)} />
              <Line label="Total Housing Payment" value={money(numbers.totalHousingPayment)} strong />
              <Line label="Front-End DTI" value={percent(numbers.frontEndDti)} />
              <Line label="Back-End DTI" value={percent(numbers.backEndDti)} />
            </div>
          </div>
          <div className="mt-6 rounded-2xl border p-5">
            <h3 className="mb-3 text-lg font-semibold">Cash to Close Breakdown</h3>
            <div className="grid grid-cols-1 gap-4 md:grid-cols-3">
              <Line label="Down Payment" value={money(numbers.downPayment)} />
              <Line label="Closing Costs" value={money(numbers.closingCosts)} />
              <Line label="Prepaids / Escrows" value={money(numbers.prepaidEscrows)} />
              <Line label="Seller Credits" value={money(numbers.sellerCredits)} />
              <Line label="Lender Credits" value={money(numbers.lenderCredits)} />
              <Line label="Other Credits" value={money(numbers.otherCredits)} />
            </div>
            <div className="mt-4 border-t pt-4"><Line label="Estimated Cash to Close" value={money(numbers.cashToClose)} strong /></div>
          </div>
          <div className="mt-6 rounded-2xl bg-slate-50 p-5 text-xs leading-5 text-slate-600"><strong>Important Disclaimer:</strong> {complianceDisclaimer}</div>
        </section>

        <div className="no-print flex flex-col gap-3 md:flex-row md:justify-end">
          <button className="rounded-2xl border border-slate-300 bg-white px-5 py-3 text-sm font-semibold text-slate-900 hover:bg-slate-100" type="button" onClick={copySummary}>Copy Client Summary</button>
          <button className="rounded-2xl bg-slate-900 px-5 py-3 text-sm font-semibold text-white hover:bg-slate-700" type="button" onClick={printEstimate}>Download / Print Client Estimate</button>
        </div>
        {copyStatus && <p className="no-print text-right text-sm text-slate-500">{copyStatus}</p>}

        <section className="no-print grid grid-cols-1 gap-6 lg:grid-cols-3">
          <div className="rounded-3xl border bg-white p-6 shadow-sm">
            <h2 className="mb-4 text-xl font-semibold">Payment & DTI Estimate</h2>
            <Line label="Principal & Interest" value={money(numbers.principalAndInterest)} />
            <Line label="Monthly Taxes" value={money(numbers.taxes)} />
            <Line label="Monthly Insurance" value={money(numbers.insurance)} />
            <Line label="HOA" value={money(numbers.hoa)} />
            <Line label="Monthly Mortgage Insurance" value={money(numbers.monthlyMortgageInsurance)} />
            <div className="mt-4 border-t pt-4">
              <Line label="Total Housing Payment" value={money(numbers.totalHousingPayment)} strong />
              <Line label="Other Monthly Debts" value={money(toNumber(scenario.otherMonthlyDebt))} />
              <Line label="Front-End DTI" value={percent(numbers.frontEndDti)} strong />
              <Line label="Back-End DTI" value={percent(numbers.backEndDti)} strong />
              <Line label="Estimated Equity" value={money(numbers.estimatedEquity)} strong />
            </div>
          </div>
          <div className="rounded-3xl border bg-white p-6 shadow-sm">
            <h2 className="mb-4 text-xl font-semibold">Pricing Notes</h2>
            <div className="space-y-3 text-sm text-slate-700">
              {alerts.length > 0 ? alerts.map((alert) => <div key={alert} className="rounded-2xl border border-amber-100 bg-amber-50 p-3 text-amber-900">{alert}</div>) : <div className="rounded-2xl border border-emerald-100 bg-emerald-50 p-3 text-emerald-900">No major pricing flags based on the current inputs.</div>}
            </div>
          </div>
          <div className="rounded-3xl border bg-white p-6 shadow-sm">
            <h2 className="mb-4 text-xl font-semibold">Client Summary</h2>
            <textarea className="h-72 w-full rounded-2xl border p-3 text-sm text-slate-700" value={clientSummary} readOnly />
          </div>
        </section>

        <section className="no-print rounded-3xl border bg-white p-6 shadow-sm">
          <div className="flex flex-col gap-3 md:flex-row md:items-center md:justify-between">
            <div>
              <h2 className="text-xl font-semibold">Calculation Checks</h2>
              <p className="mt-1 text-sm text-slate-500">{testsPassed ? "All calculation checks passed." : "One or more calculation checks failed."}</p>
            </div>
            <span className={`rounded-full px-4 py-2 text-sm font-semibold ${testsPassed ? "bg-emerald-50 text-emerald-700" : "bg-red-50 text-red-700"}`}>{tests.filter((test) => test.pass).length}/{tests.length} passed</span>
          </div>
          <div className="mt-4 grid grid-cols-1 gap-3 md:grid-cols-2">
            {tests.map((test) => <div key={test.name} className={`rounded-2xl border p-3 text-sm ${test.pass ? "border-emerald-100 bg-emerald-50 text-emerald-900" : "border-red-100 bg-red-50 text-red-900"}`}>{test.pass ? "Pass" : "Fail"}: {test.name}</div>)}
          </div>
        </section>

        <div className="rounded-3xl bg-slate-900 p-5 text-sm text-slate-100">Compliance reminder: This tool is for preliminary estimates only. Final pricing must be confirmed through the lender pricing engine, AUS, underwriting, disclosures, and current investor guidelines.</div>
      </div>
    </main>
  );
}

function TextField({ label, value, onChange, type = "text" }) {
  return <label className="block"><span className="text-sm font-medium text-slate-700">{label}</span><input className="mt-1 w-full rounded-xl border border-slate-300 bg-white px-3 py-2 text-sm text-slate-900 outline-none focus:border-slate-500 focus:ring-2 focus:ring-slate-200" type={type} value={value} onChange={(event) => onChange(event.target.value)} /></label>;
}

function SelectField({ label, value, onChange, options }) {
  return <label className="block"><span className="text-sm font-medium text-slate-700">{label}</span><select className="mt-1 w-full rounded-xl border border-slate-300 bg-white px-3 py-2 text-sm text-slate-900 outline-none focus:border-slate-500 focus:ring-2 focus:ring-slate-200" value={value} onChange={(event) => onChange(event.target.value)}>{options.map((option) => <option key={option.value} value={option.value}>{option.label}</option>)}</select></label>;
}

function NumberField({ label, value, onChange }) {
  return <label className="block"><span className="text-sm font-medium text-slate-700">{label}</span><input className="mt-1 w-full rounded-xl border border-slate-300 bg-white px-3 py-2 text-sm text-slate-900 outline-none focus:border-slate-500 focus:ring-2 focus:ring-slate-200" type="number" min="0" step="any" value={value} onChange={(event) => onChange(event.target.value)} /></label>;
}

function EstimateBox({ label, value }) {
  return <div className="rounded-2xl border bg-slate-50 p-4"><p className="text-xs font-semibold uppercase tracking-wide text-slate-500">{label}</p><p className="mt-2 text-xl font-bold text-slate-900">{value}</p></div>;
}

function MetricCard({ title, value, subtitle }) {
  return <div className="rounded-3xl border bg-white p-6 shadow-sm"><p className="text-sm font-medium text-slate-500">{title}</p><p className="mt-3 text-3xl font-bold text-slate-900">{value}</p><p className="mt-2 text-sm text-slate-500">{subtitle}</p></div>;
}

function MetricInline({ label, value }) {
  return <div className="rounded-2xl border bg-slate-50 p-4"><p className="text-sm font-medium text-slate-500">{label}</p><p className="mt-1 text-xl font-bold text-slate-900">{value}</p></div>;
}

function Line({ label, value, strong = false }) {
  return <div className={`flex justify-between gap-3 py-2 ${strong ? "font-semibold text-slate-900" : "text-slate-700"}`}><span>{label}</span><span>{value}</span></div>;
}
# loan-pricing-app
