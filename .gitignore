from __future__ import annotations

import math
from dataclasses import asdict, dataclass
from typing import Any, Literal

from fastapi import FastAPI
from fastapi.responses import HTMLResponse, JSONResponse
from pydantic import BaseModel, Field
import uvicorn


# =========================
# 型定義
# =========================
MortgageType = Literal["equal_payment", "equal_principal"]
PropertyType = Literal["new", "used"]
EnergyClass = Literal["certified", "zeh", "energy_standard", "other"]


# =========================
# リクエスト / レスポンスモデル
# =========================
class LoanRequest(BaseModel):
    annual_income: int = Field(..., gt=0, description="年収（円）")
    loan_amount: int = Field(..., gt=0, description="借入金額（円）")
    annual_interest_rate: float = Field(..., ge=0, le=20, description="年利（%）")
    years: int = Field(..., ge=1, le=50, description="返済年数")
    bonus_payment_per_year: int = Field(0, ge=0, description="年間ボーナス返済額（円）")
    mortgage_type: MortgageType = Field("equal_payment", description="返済方式")

    move_in_year: int = Field(2026, ge=2022, le=2035, description="入居年")
    property_type: PropertyType = Field("new", description="物件種別")
    energy_class: EnergyClass = Field("zeh", description="省エネ区分")
    social_insurance_rate: float = Field(0.15, ge=0, le=0.3, description="社会保険料率の簡易設定")
    other_deductions: int = Field(0, ge=0, description="その他控除額（円）")


@dataclass
class YearlyDetail:
    year: int
    annual_payment: int
    annual_principal: int
    annual_interest: int
    ending_balance: int
    housing_credit_cap: int
    income_tax_credit_used: int
    resident_tax_credit_used: int
    total_tax_credit_used: int
    unrecovered_credit: int


@dataclass
class Recommendation:
    recommended: bool
    reason: str
    front_end_dti_percent: float


@dataclass
class LoanResponse:
    summary: dict[str, Any]
    recommendation: Recommendation
    yearly_details: list[YearlyDetail]

    def to_dict(self) -> dict[str, Any]:
        return {
            "summary": self.summary,
            "recommendation": asdict(self.recommendation),
            "yearly_details": [asdict(item) for item in self.yearly_details],
        }


# =========================
# 税額・控除関連の簡易ロジック
# =========================
def employment_income_deduction(salary_income: int) -> int:
    if salary_income <= 1_900_000:
        return 650_000
    if salary_income <= 3_600_000:
        return int(salary_income * 0.30 + 80_000)
    if salary_income <= 6_600_000:
        return int(salary_income * 0.20 + 440_000)
    if salary_income <= 8_500_000:
        return int(salary_income * 0.10 + 1_100_000)
    return 1_950_000


def basic_deduction(total_income: int) -> int:
    if total_income <= 1_320_000:
        return 950_000
    if total_income <= 3_360_000:
        return 580_000
    if total_income <= 4_890_000:
        return 680_000
    if total_income <= 6_550_000:
        return 630_000
    if total_income <= 23_500_000:
        return 580_000
    if total_income <= 24_000_000:
        return 480_000
    if total_income <= 24_500_000:
        return 320_000
    if total_income <= 25_000_000:
        return 160_000
    return 0


def income_tax_amount(taxable_income: int) -> int:
    taxable_income = max(0, taxable_income // 1000 * 1000)
    if taxable_income <= 1_949_000:
        base = taxable_income * 0.05
    elif taxable_income <= 3_299_000:
        base = taxable_income * 0.10 - 97_500
    elif taxable_income <= 6_949_000:
        base = taxable_income * 0.20 - 427_500
    elif taxable_income <= 8_999_000:
        base = taxable_income * 0.23 - 636_000
    elif taxable_income <= 17_999_000:
        base = taxable_income * 0.33 - 1_536_000
    elif taxable_income <= 39_999_000:
        base = taxable_income * 0.40 - 2_796_000
    else:
        base = taxable_income * 0.45 - 4_796_000
    return max(0, int(base * 1.021))


def resident_tax_amount(taxable_income: int) -> int:
    return max(0, int(taxable_income * 0.10))


def get_housing_credit_settings(move_in_year: int, property_type: PropertyType, energy_class: EnergyClass) -> tuple[int, float, int]:
    credit_rate = 0.007

    if move_in_year < 2026:
        if property_type == "new":
            caps = {
                "certified": 45_000_000,
                "zeh": 35_000_000,
                "energy_standard": 30_000_000,
                "other": 0,
            }
            return caps[energy_class], credit_rate, 13
        caps = {
            "certified": 30_000_000,
            "zeh": 30_000_000,
            "energy_standard": 30_000_000,
            "other": 20_000_000,
        }
        return caps[energy_class], credit_rate, 10

    if property_type == "new":
        caps = {
            "certified": 45_000_000,
            "zeh": 35_000_000,
            "energy_standard": 20_000_000 if move_in_year in (2026, 2027) else 0,
            "other": 0,
        }
        return caps[energy_class], credit_rate, 13

    caps = {
        "certified": 35_000_000,
        "zeh": 30_000_000,
        "energy_standard": 30_000_000,
        "other": 20_000_000,
    }
    return caps[energy_class], credit_rate, 10


# =========================
# 計算ロジック
# =========================
def annuity_payment(principal: int, annual_rate_percent: float, months: int) -> int:
    if months <= 0:
        raise ValueError("months must be > 0")

    monthly_rate = annual_rate_percent / 100 / 12
    if monthly_rate == 0:
        return int(round(principal / months))

    payment = principal * monthly_rate / (1 - math.pow(1 + monthly_rate, -months))
    return int(round(payment))


def calculate_loan(req: LoanRequest) -> LoanResponse:
    months = req.years * 12
    monthly_rate = req.annual_interest_rate / 100 / 12
    monthly_payment = annuity_payment(req.loan_amount, req.annual_interest_rate, months)

    salary_deduction = employment_income_deduction(req.annual_income)
    salary_income = max(0, req.annual_income - salary_deduction)
    total_income = salary_income
    base_ded = basic_deduction(total_income)
    social_insurance = int(req.annual_income * req.social_insurance_rate)
    taxable_income = max(0, total_income - base_ded - social_insurance - req.other_deductions)
    resident_taxable_income = max(0, total_income - 430_000 - social_insurance - req.other_deductions)
    estimated_income_tax = income_tax_amount(taxable_income)
    estimated_resident_tax = resident_tax_amount(resident_taxable_income)
    resident_credit_cap = min(int(taxable_income * 0.05), 97_500)

    credit_cap_balance, credit_rate, credit_years = get_housing_credit_settings(
        req.move_in_year, req.property_type, req.energy_class
    )

    remaining_balance = req.loan_amount
    yearly_details: list[YearlyDetail] = []
    total_payment = 0
    total_interest = 0
    total_tax_credit_used_sum = 0

    equal_principal_base = req.loan_amount // months if req.mortgage_type == "equal_principal" else 0
    principal_remainder = req.loan_amount % months if req.mortgage_type == "equal_principal" else 0

    for year in range(1, req.years + 1):
        annual_payment = 0
        annual_principal = 0
        annual_interest = 0

        for _ in range(12):
            if remaining_balance <= 0:
                break

            interest = int(round(remaining_balance * monthly_rate))

            if req.mortgage_type == "equal_principal":
                principal_part = equal_principal_base
                if principal_remainder > 0:
                    principal_part += 1
                    principal_remainder -= 1
                principal_part = min(principal_part, remaining_balance)
                payment = principal_part + interest
            else:
                payment = monthly_payment
                principal_part = payment - interest
                if principal_part > remaining_balance:
                    principal_part = remaining_balance
                    payment = principal_part + interest

            remaining_balance -= principal_part
            annual_payment += payment
            annual_principal += principal_part
            annual_interest += interest

        if req.bonus_payment_per_year > 0 and remaining_balance > 0:
            bonus = min(req.bonus_payment_per_year, remaining_balance)
            remaining_balance -= bonus
            annual_payment += bonus
            annual_principal += bonus

        total_payment += annual_payment
        total_interest += annual_interest

        housing_credit_cap = 0
        income_tax_credit_used = 0
        resident_tax_credit_used = 0
        total_tax_credit_used = 0
        unrecovered_credit = 0

        if year <= credit_years and credit_cap_balance > 0:
            eligible_balance = min(remaining_balance, credit_cap_balance)
            housing_credit_cap = int(eligible_balance * credit_rate)
            income_tax_credit_used = min(estimated_income_tax, housing_credit_cap)
            remaining_credit = housing_credit_cap - income_tax_credit_used
            resident_tax_credit_used = min(resident_credit_cap, remaining_credit)
            total_tax_credit_used = income_tax_credit_used + resident_tax_credit_used
            unrecovered_credit = housing_credit_cap - total_tax_credit_used

        total_tax_credit_used_sum += total_tax_credit_used

        yearly_details.append(
            YearlyDetail(
                year=year,
                annual_payment=annual_payment,
                annual_principal=annual_principal,
                annual_interest=annual_interest,
                ending_balance=max(0, remaining_balance),
                housing_credit_cap=housing_credit_cap,
                income_tax_credit_used=income_tax_credit_used,
                resident_tax_credit_used=resident_tax_credit_used,
                total_tax_credit_used=total_tax_credit_used,
                unrecovered_credit=unrecovered_credit,
            )
        )

        if remaining_balance <= 0:
            break

    first_year_payment = yearly_details[0].annual_payment if yearly_details else 0
    first_year_credit = yearly_details[0].total_tax_credit_used if yearly_details else 0
    dti = first_year_payment / req.annual_income * 100
    first_year_net_burden = first_year_payment - first_year_credit

    if dti <= 20:
        recommended = True
        reason = "返済負担率は比較的低く、無理のない水準です。"
    elif dti <= 25:
        recommended = True
        reason = "一般的には許容範囲ですが、手元資金も確認するのがおすすめです。"
    elif dti <= 30:
        recommended = False
        reason = "やや負担が重めです。借入金額や年数の見直しをおすすめします。"
    else:
        recommended = False
        reason = "返済負担率が高く、慎重な判断が必要です。"

    return LoanResponse(
        summary={
            "月々返済額": monthly_payment,
            "初年度返済額": first_year_payment,
            "総支払額": total_payment,
            "総利息": total_interest,
            "返済負担率(%)": round(dti, 2),
            "推定課税所得": taxable_income,
            "推定所得税": estimated_income_tax,
            "推定住民税": estimated_resident_tax,
            "住宅ローン控除率": credit_rate,
            "住宅ローン控除期間": credit_years,
            "控除対象借入上限": credit_cap_balance,
            "初年度控除見込額": first_year_credit,
            "初年度実質負担額": first_year_net_burden,
            "累計控除見込額": total_tax_credit_used_sum,
        },
        recommendation=Recommendation(
            recommended=recommended,
            reason=reason,
            front_end_dti_percent=round(dti, 2),
        ),
        yearly_details=yearly_details,
    )


# =========================
# FastAPI アプリ
# =========================
app = FastAPI(title="住宅ローン計算ツール", version="2.0.0")


HTML_PAGE = """
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>住宅ローン計算ツール</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    body {
      font-family: Arial, sans-serif;
      margin: 0;
      background: #f6f8fb;
      color: #1f2937;
    }
    .container {
      max-width: 1280px;
      margin: 0 auto;
      padding: 24px;
    }
    h1 {
      margin-bottom: 8px;
    }
    .sub {
      color: #6b7280;
      margin-bottom: 24px;
      line-height: 1.6;
    }
    .grid {
      display: grid;
      grid-template-columns: 380px 1fr;
      gap: 20px;
    }
    .card {
      background: white;
      border-radius: 16px;
      padding: 20px;
      box-shadow: 0 2px 10px rgba(0,0,0,0.05);
    }
    .form-row {
      margin-bottom: 14px;
    }
    label {
      display: block;
      font-size: 14px;
      margin-bottom: 6px;
      color: #374151;
    }
    input, select, button {
      width: 100%;
      box-sizing: border-box;
      padding: 10px 12px;
      border-radius: 10px;
      border: 1px solid #d1d5db;
      font-size: 14px;
    }
    button {
      background: #111827;
      color: white;
      border: none;
      cursor: pointer;
      font-weight: 600;
    }
    button:hover {
      opacity: 0.95;
    }
    .summary-grid {
      display: grid;
      grid-template-columns: repeat(3, 1fr);
      gap: 14px;
      margin-bottom: 20px;
    }
    .metric {
      background: #f9fafb;
      border-radius: 14px;
      padding: 16px;
    }
    .metric-title {
      color: #6b7280;
      font-size: 13px;
      margin-bottom: 6px;
    }
    .metric-value {
      font-size: 24px;
      font-weight: 700;
    }
    .metric-sub {
      margin-top: 6px;
      font-size: 12px;
      color: #6b7280;
    }
    .recommend {
      padding: 16px;
      border-radius: 14px;
      margin-bottom: 20px;
    }
    .recommend.ok {
      background: #ecfdf5;
      color: #065f46;
    }
    .recommend.warn {
      background: #fff7ed;
      color: #9a3412;
    }
    .chart-wrap {
      margin-top: 16px;
    }
    table {
      width: 100%;
      border-collapse: collapse;
      margin-top: 18px;
      font-size: 14px;
    }
    th, td {
      padding: 10px 8px;
      border-bottom: 1px solid #e5e7eb;
      text-align: left;
      vertical-align: top;
    }
    .muted {
      color: #6b7280;
      font-size: 13px;
      line-height: 1.6;
    }
    .section-title {
      margin-top: 20px;
      margin-bottom: 8px;
      font-size: 16px;
      font-weight: 700;
    }
    @media (max-width: 1000px) {
      .grid {
        grid-template-columns: 1fr;
      }
      .summary-grid {
        grid-template-columns: repeat(2, 1fr);
      }
    }
    @media (max-width: 720px) {
      .summary-grid {
        grid-template-columns: 1fr;
      }
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>住宅ローン計算ツール</h1>
    <div class="sub">
      通常の返済試算に加えて、住宅ローン控除・所得税・住民税の簡易試算も表示します。<br />
      同一ネットワーク内のユーザーも、このPCのIPアドレスでアクセスできます。
    </div>

    <div class="grid">
      <div class="card">
        <h3>入力条件</h3>
        <div class="form-row">
          <label for="annual_income">年収（円）</label>
          <input id="annual_income" value="7500000" />
        </div>
        <div class="form-row">
          <label for="loan_amount">借入金額（円）</label>
          <input id="loan_amount" value="60000000" />
        </div>
        <div class="form-row">
          <label for="annual_interest_rate">年利（%）</label>
          <input id="annual_interest_rate" value="0.6" />
        </div>
        <div class="form-row">
          <label for="years">返済年数</label>
          <input id="years" value="35" />
        </div>
        <div class="form-row">
          <label for="bonus_payment_per_year">年間ボーナス返済額（円）</label>
          <input id="bonus_payment_per_year" value="0" />
        </div>
        <div class="form-row">
          <label for="mortgage_type">返済方式</label>
          <select id="mortgage_type">
            <option value="equal_payment">元利均等返済</option>
            <option value="equal_principal">元金均等返済</option>
          </select>
        </div>

        <div class="section-title">住宅ローン控除の条件</div>
        <div class="form-row">
          <label for="move_in_year">入居年</label>
          <input id="move_in_year" value="2026" />
        </div>
        <div class="form-row">
          <label for="property_type">物件種別</label>
          <select id="property_type">
            <option value="new">新築</option>
            <option value="used">中古</option>
          </select>
        </div>
        <div class="form-row">
          <label for="energy_class">省エネ区分</label>
          <select id="energy_class">
            <option value="certified">認定住宅</option>
            <option value="zeh" selected>ZEH水準省エネ住宅</option>
            <option value="energy_standard">省エネ基準適合住宅</option>
            <option value="other">その他</option>
          </select>
        </div>
        <div class="form-row">
          <label for="social_insurance_rate">社会保険料率（簡易）</label>
          <input id="social_insurance_rate" value="0.15" />
        </div>
        <div class="form-row">
          <label for="other_deductions">その他控除額（円）</label>
          <input id="other_deductions" value="0" />
        </div>

        <button onclick="calculateLoan()">計算する</button>
        <p class="muted" style="margin-top: 12px;">
          例：同一ネットワークの他ユーザーは http://あなたのPCのIP:8000 でアクセスします。<br />
          税額と控除は簡易試算であり、最終判断には実際の税務条件をご確認ください。
        </p>
      </div>

      <div class="card">
        <h3>計算結果</h3>
        <div class="summary-grid">
          <div class="metric">
            <div class="metric-title">月々返済額</div>
            <div class="metric-value" id="monthly_payment">-</div>
          </div>
          <div class="metric">
            <div class="metric-title">初年度返済額</div>
            <div class="metric-value" id="first_year_payment">-</div>
          </div>
          <div class="metric">
            <div class="metric-title">初年度控除見込額</div>
            <div class="metric-value" id="first_year_credit">-</div>
            <div class="metric-sub">所得税 + 住民税の簡易試算</div>
          </div>
          <div class="metric">
            <div class="metric-title">初年度実質負担額</div>
            <div class="metric-value" id="first_year_net">-</div>
          </div>
          <div class="metric">
            <div class="metric-title">総支払額</div>
            <div class="metric-value" id="total_payment">-</div>
          </div>
          <div class="metric">
            <div class="metric-title">総利息</div>
            <div class="metric-value" id="total_interest">-</div>
          </div>
          <div class="metric">
            <div class="metric-title">推定所得税</div>
            <div class="metric-value" id="income_tax">-</div>
          </div>
          <div class="metric">
            <div class="metric-title">推定住民税</div>
            <div class="metric-value" id="resident_tax">-</div>
          </div>
          <div class="metric">
            <div class="metric-title">累計控除見込額</div>
            <div class="metric-value" id="total_credit_sum">-</div>
          </div>
        </div>

        <div id="recommend_box" class="recommend warn">
          <div><strong>おすすめ判定</strong></div>
          <div id="recommend_text">-</div>
          <div id="dti_text" style="margin-top: 6px;">返済負担率：-</div>
        </div>

        <div class="chart-wrap">
          <canvas id="balanceChart" height="110"></canvas>
        </div>

        <table>
          <thead>
            <tr>
              <th>年</th>
              <th>年間返済額</th>
              <th>利息</th>
              <th>年末残高</th>
              <th>控除見込額</th>
              <th>未回収控除</th>
            </tr>
          </thead>
          <tbody id="details_body"></tbody>
        </table>
      </div>
    </div>
  </div>

  <script>
    let balanceChart = null;

    function yen(value) {
      return '¥' + Math.round(value).toLocaleString('ja-JP');
    }

    async function calculateLoan() {
      const payload = {
        annual_income: Number(document.getElementById('annual_income').value),
        loan_amount: Number(document.getElementById('loan_amount').value),
        annual_interest_rate: Number(document.getElementById('annual_interest_rate').value),
        years: Number(document.getElementById('years').value),
        bonus_payment_per_year: Number(document.getElementById('bonus_payment_per_year').value),
        mortgage_type: document.getElementById('mortgage_type').value,
        move_in_year: Number(document.getElementById('move_in_year').value),
        property_type: document.getElementById('property_type').value,
        energy_class: document.getElementById('energy_class').value,
        social_insurance_rate: Number(document.getElementById('social_insurance_rate').value),
        other_deductions: Number(document.getElementById('other_deductions').value),
      };

      const response = await fetch('/api/calculate', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload),
      });

      const data = await response.json();
      if (!response.ok) {
        alert(data.detail || '計算時にエラーが発生しました');
        return;
      }

      document.getElementById('monthly_payment').textContent = yen(data.summary['月々返済額']);
      document.getElementById('first_year_payment').textContent = yen(data.summary['初年度返済額']);
      document.getElementById('first_year_credit').textContent = yen(data.summary['初年度控除見込額']);
      document.getElementById('first_year_net').textContent = yen(data.summary['初年度実質負担額']);
      document.getElementById('total_payment').textContent = yen(data.summary['総支払額']);
      document.getElementById('total_interest').textContent = yen(data.summary['総利息']);
      document.getElementById('income_tax').textContent = yen(data.summary['推定所得税']);
      document.getElementById('resident_tax').textContent = yen(data.summary['推定住民税']);
      document.getElementById('total_credit_sum').textContent = yen(data.summary['累計控除見込額']);

      document.getElementById('recommend_text').textContent = data.recommendation.reason;
      document.getElementById('dti_text').textContent = `返済負担率：${data.recommendation.front_end_dti_percent}%`;

      const box = document.getElementById('recommend_box');
      box.className = data.recommendation.recommended ? 'recommend ok' : 'recommend warn';

      const tbody = document.getElementById('details_body');
      tbody.innerHTML = '';
      data.yearly_details.slice(0, 12).forEach((row) => {
        const tr = document.createElement('tr');
        tr.innerHTML = `
          <td>${row.year}年目</td>
          <td>${yen(row.annual_payment)}</td>
          <td>${yen(row.annual_interest)}</td>
          <td>${yen(row.ending_balance)}</td>
          <td>${yen(row.total_tax_credit_used)}</td>
          <td>${yen(row.unrecovered_credit)}</td>
        `;
        tbody.appendChild(tr);
      });

      const labels = data.yearly_details.map((row) => `${row.year}年目`);
      const balances = data.yearly_details.map((row) => row.ending_balance);
      const credits = data.yearly_details.map((row) => row.total_tax_credit_used);

      const ctx = document.getElementById('balanceChart').getContext('2d');
      if (balanceChart) {
        balanceChart.destroy();
      }

      balanceChart = new Chart(ctx, {
        type: 'line',
        data: {
          labels,
          datasets: [
            {
              label: '年末残高',
              data: balances,
              tension: 0.25,
              borderWidth: 2,
              fill: false,
              yAxisID: 'y',
            },
            {
              label: '控除見込額',
              data: credits,
              tension: 0.25,
              borderWidth: 2,
              fill: false,
              yAxisID: 'y1',
            }
          ]
        },
        options: {
          responsive: true,
          interaction: { mode: 'index', intersect: false },
          plugins: {
            legend: { display: true }
          },
          scales: {
            y: {
              position: 'left'
            },
            y1: {
              position: 'right',
              grid: { drawOnChartArea: false }
            }
          }
        }
      });
    }

    calculateLoan();
  </script>
</body>
</html>
"""


@app.get("/", response_class=HTMLResponse)
def index() -> HTMLResponse:
    return HTMLResponse(content=HTML_PAGE)


@app.get("/health")
def health() -> dict[str, bool]:
    return {"ok": True}


@app.post("/api/calculate")
def api_calculate(req: LoanRequest) -> JSONResponse:
    result = calculate_loan(req)
    return JSONResponse(content=result.to_dict())


if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
