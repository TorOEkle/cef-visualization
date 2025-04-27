<script lang="ts">
  import { onMount } from 'svelte';
  
  // Constants for visualization
  const margin = { top: 50, right: 50, bottom: 50, left: 70 };
  const width = 800 - margin.left - margin.right;
  const height = 500 - margin.top - margin.bottom;
  
  // Define interfaces for our data structures
  interface DataPoint {
    years_education: number;
    log_weekly_earnings: number;
  }
  
  interface CEFPoint {
    years_education: number;
    avg_log_weekly_earnings: number;
  }
  
  interface PDFPoint {
    value: number;
    density: number;
  }
  
  interface PDFData {
    [key: number]: PDFPoint[];
  }
  
  // Data state
  let data: DataPoint[] = [];
  let cefData: CEFPoint[] = [];
  let pdfData: PDFData = {};
  
  // Axis ranges (will be set dynamically)
  let xMin: number = 0, xMax: number = 0, yMin: number = 0, yMax: number = 0;
  
  // Selected years for which to display PDFs
  const selectedYears: number[] = [8, 12, 16, 20];
  
  // Function to calculate the CEF (conditional expectation function)
  function calculateCEF(data: DataPoint[]): CEFPoint[] {
    // Group data by years of education
    const grouped: {[key: number]: number[]} = {};
    data.forEach(d => {
      if (!grouped[d.years_education]) {
        grouped[d.years_education] = [];
      }
      grouped[d.years_education].push(d.log_weekly_earnings);
    });
    
    // Calculate average for each education level
    return Object.keys(grouped).map(year => {
      const earnings = grouped[parseInt(year)];
      const avgEarning = earnings.reduce((sum: number, val: number) => sum + val, 0) / earnings.length;
      return {
        years_education: parseInt(year),
        avg_log_weekly_earnings: avgEarning
      };
    }).sort((a, b) => a.years_education - b.years_education);
  }
  
  // Function to calculate axis ranges from data
  function calculateAxisRanges(data: DataPoint[], cefData: CEFPoint[]): void {
    // Find min and max of years of education
    const eduValues = data.map(d => d.years_education);
    xMin = Math.floor(Math.min(...eduValues)) - 1;
    xMax = Math.ceil(Math.max(...eduValues)) + 1;
    
    // Find min and max of log earnings
    const earningsValues = data.map(d => d.log_weekly_earnings);
    yMin = Math.floor(Math.min(...earningsValues) * 10) / 10 - 0.5;
    yMax = Math.ceil(Math.max(...earningsValues) * 10) / 10 + 0.5;
    
    // Generate tick values for X and Y axes - ensure even numbers only for X axis
    xTicks = [];
    // Start from the first even number after or at xMin
    let start = xMin % 2 === 0 ? xMin : xMin + 1;
    for (let i = start; i <= xMax; i += 2) {
      xTicks.push(i);
    }
    
    yTicks = Array.from(
      { length: Math.ceil((yMax - yMin) / 0.5) + 1 }, 
      (_, i) => Math.round((yMin + i * 0.5) * 10) / 10
    );
  }
  
  // Function to calculate PDFs for selected years
  function calculatePDFs(data: DataPoint[], selectedYears: number[]): PDFData {
    const pdfs: PDFData = {};
    
    selectedYears.forEach(year => {
      // Filter data for this year
      const yearData = data.filter(d => d.years_education === year)
                           .map(d => d.log_weekly_earnings);
      
      if (yearData.length === 0) return;
      
      // Find min and max for binning
      const min = Math.min(...yearData);
      const max = Math.max(...yearData);
      
      // Create bins
      const binCount = 20;
      const binWidth = (max - min) / binCount;
      const bins = Array(binCount).fill(0);
      
      // Fill bins
      yearData.forEach(value => {
        const binIndex = Math.min(
          Math.floor((value - min) / binWidth),
          binCount - 1
        );
        bins[binIndex]++;
      });
      
      // Normalize bins to create PDF
      const maxCount = Math.max(...bins);
      const normalizedBins = bins.map(count => count / maxCount);
      
      // Create PDF data points
      pdfs[year] = normalizedBins.map((density, i) => {
        return {
          value: min + i * binWidth + binWidth / 2,
          density
        };
      });
    });
    
    return pdfs;
  }
  
  // Scale functions for mapping data to SVG coordinates
  function xScale(value: number): number {
    return ((value - xMin) / (xMax - xMin)) * width;
  }
  
  function yScale(value: number): number {
    return height - ((value - yMin) / (yMax - yMin)) * height;
  }
  
  // Function to generate path data for CEF line
  function cefLine(data: CEFPoint[]): string {
    return data.map((d, i) => 
      (i === 0 ? "M" : "L") + 
      xScale(d.years_education) + "," + 
      yScale(d.avg_log_weekly_earnings)
    ).join(" ");
  }
  
  // Function to generate PDF paths
  function pdfPath(pdfPoints: PDFPoint[] | undefined, x: number, maxWidth: number = 50): string {
    if (!pdfPoints || pdfPoints.length === 0) return "";
     
    // Only draw the right half of the PDF (half violin plot)
    return pdfPoints.map((point, i) => {
      const xPos = x;
      const yPos = yScale(point.value);
      const width = point.density * maxWidth;
      
      // Start at the center line (x position)
      return (i === 0 ? "M" : "L") + xPos + "," + yPos;
    }).join(" ") + 
    // Then draw outward to the right based on density
    pdfPoints.slice().reverse().map((point, i) => {
      const xPos = x;
      const yPos = yScale(point.value);
      const width = point.density * maxWidth;
      
      // Only extend to the right side
      return "L" + (xPos + width) + "," + yPos;
    }).join(" ") + "Z";
  }
  
  // Generate X and Y axis ticks
  let xTicks: number[] = [];
  let yTicks: number[] = [];
  
  onMount(async () => {
    try {
      // Fetch and parse the CSV data
      const response = await fetch('/data.csv');
      const text = await response.text();
      
      // Simple CSV parsing (header row + data rows)
      const rows = text.trim().split('\n');
      const headers = rows[0].split(',');
      
      // Find the correct column indices
      const yearColumnIndex = headers.findIndex(h => 
        h.includes('education') || h.includes('year'));
      const earningsColumnIndex = headers.findIndex(h => 
        h.includes('earning') || h.includes('wage') || h.includes('income'));
      
      if (yearColumnIndex === -1 || earningsColumnIndex === -1) {
        console.error('Could not identify the correct columns in CSV');
        return;
      }
      
      data = rows.slice(1).map(row => {
        const values = row.split(',');
        return {
          years_education: parseInt(values[yearColumnIndex]),
          log_weekly_earnings: parseFloat(values[earningsColumnIndex])
        };
      });
      
      // Calculate CEF and PDFs
      cefData = calculateCEF(data);
      
      // Set dynamic axis ranges
      calculateAxisRanges(data, cefData);
      
      // Calculate PDFs with updated axis scales
      pdfData = calculatePDFs(data, selectedYears);
      
    } catch (error) {
      console.error('Error loading or processing data:', error);
    }
  });
</script>

<svelte:head>
  <title>CEF Visualization</title>
</svelte:head>

<main>
  <h1>Conditional Expectation Function (CEF) Visualization</h1>
  <p>Relationship between years of education and log weekly earnings</p>
  
  <div class="chart-container">
    <svg width={width + margin.left + margin.right} height={height + margin.top + margin.bottom}>
      <g transform="translate({margin.left},{margin.top})">
        <!-- X and Y axes -->
        <line class="axis" x1="0" y1={height} x2={width} y2={height}></line>
        <line class="axis" x1="0" y1="0" x2="0" y2={height}></line>
        
        <!-- X-axis ticks and labels -->
        {#each xTicks as year}
          <line 
            class="tick" 
            x1={xScale(year)} 
            y1={height} 
            x2={xScale(year)} 
            y2={height + 5}
          ></line>
          <text 
            class="axis-label" 
            x={xScale(year)} 
            y={height + 20} 
            text-anchor="middle"
          >{year}</text>
        {/each}
        
        <!-- Y-axis ticks and labels -->
        {#each yTicks as earning}
          <line 
            class="tick" 
            x1="0" 
            y1={yScale(earning)} 
            x2="-5" 
            y2={yScale(earning)}
          ></line>
          <text 
            class="axis-label" 
            x="-10" 
            y={yScale(earning)} 
            text-anchor="end" 
            dominant-baseline="middle"
          >{earning}</text>
        {/each}
        
        <!-- Axis titles -->
        <text 
          class="axis-title" 
          x={width / 2} 
          y={height + 40} 
          text-anchor="middle"
        >Years of Education</text>
        
        <text 
          class="axis-title" 
          x={-40} 
          y={height / 2} 
          text-anchor="middle" 
          transform="rotate(-90, {-40}, {height / 2})"
        >Log Weekly Earnings</text>
        
        <!-- Raw data points (small and light opacity) -->
        {#if false} <!-- Removing individual data points as requested -->
          {#each data as point}
            <circle 
              cx={xScale(point.years_education)} 
              cy={yScale(point.log_weekly_earnings)} 
              r="2" 
              class="data-point"
            ></circle>
          {/each}
        {/if}
        
        <!-- CEF line -->
        {#if cefData.length > 0}
          <path 
            d={cefLine(cefData)} 
            class="cef-line"
          ></path>
          
          <!-- Average value points (large red dots) -->
          {#each cefData as point}
            <circle 
              cx={xScale(point.years_education)} 
              cy={yScale(point.avg_log_weekly_earnings)} 
              r="5" 
              class="average-point"
            ></circle>
          {/each}
        {/if}
        
        <!-- PDFs for selected years -->
        {#each selectedYears as year}
          {#if pdfData[year]}
            <path 
              d={pdfPath(pdfData[year], xScale(year))} 
              class="pdf" 
              fill-opacity="0.3"
            ></path>
          {/if}
        {/each}
      </g>
    </svg>
  </div>
</main>

<style>
  main {
    max-width: 900px;
    margin: 0 auto;
    padding: 20px;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  }
  
  h1 {
    color: #333;
    font-size: 1.8rem;
    margin-bottom: 0.5rem;
  }
  
  p {
    color: #666;
    margin-bottom: 2rem;
  }
  
  .chart-container {
    margin: 20px 0;
    background-color: #fff;
    border-radius: 5px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
    padding: 20px;
    overflow-x: auto;
  }
  
  svg {
    background-color: #fff;
  }
  
  .axis {
    stroke: #333;
    stroke-width: 1;
  }
  
  .tick {
    stroke: #333;
    stroke-width: 1;
  }
  
  .axis-label {
    fill: #333;
    font-size: 12px;
  }
  
  .axis-title {
    fill: #333;
    font-size: 14px;
    font-weight: bold;
  }
  
  .data-point {
    fill: rgba(70, 130, 180, 0.5);
    stroke: steelblue;
    stroke-width: 0.5;
  }
  
  .average-point {
    fill: #ff3e00;
    stroke: #990000;
    stroke-width: 1;
  }
  
  .cef-line {
    fill: none;
    stroke: #0059ff;
    stroke-width: 2;
  }
  
  .pdf {
    fill: #4682b4;
    stroke: #2b506e;
    stroke-width: 1;
  }
  
  .pdf-label {
    fill: #333;
    font-size: 10px;
  }
</style>
