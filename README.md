<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Microplastic Dataset Generator</title>
    <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body>
    <div id="root"></div>
    <script type="text/babel">
const { useState } = React;

const App = () => {
  const [generating, setGenerating] = useState(false);
  const [dataset, setDataset] = useState(null);
  const [filtrationDataset, setFiltrationDataset] = useState(null);
  const [progress, setProgress] = useState(0);

  const polymerProps = {
    PE: { qy: 0.38, be: 0.85, bf: 1200 },
    PP: { qy: 0.35, be: 0.80, bf: 1100 },
    PS: { qy: 0.42, be: 0.90, bf: 1400 },
    PET: { qy: 0.33, be: 0.75, bf: 1000 }
  };

  const genSizes = (n) => {
    const sizes = [];
    for (let i = 0; i < n; i++) {
      const u1 = Math.random(), u2 = Math.random();
      const z = Math.sqrt(-2 * Math.log(u1)) * Math.cos(2 * Math.PI * u2);
      sizes.push(Math.max(5, Math.min(5000, Math.exp(Math.log(500) + 0.6 * z))));
    }
    return sizes;
  };

  const calcSignal = (sizes, poly, nrConc, exWl) => {
    const p = polymerProps[poly];
    const cf = nrConc / (nrConc + 2);
    const wf = Math.exp(-Math.pow(exWl - 550, 2) / 3200);
    let sig = 0;
    sizes.forEach(s => {
      const ri = Math.pow(s / 500, 2) * (0.85 + Math.random() * 0.3);
      sig += p.bf * p.qy * p.be * ri * cf * wf * 0.4e-3;
    });
    return sig;
  };

  const addNoise = (s) => s + (Math.random() - 0.5) * 0.003 * Math.sqrt(Math.abs(s)) + (Math.random() - 0.5) * 0.003;

  const digitize = (v) => Math.round(Math.max(0, Math.min(5, v)) / 5 * 4095);

  const genMeas = (n, poly, nr, ex) => {
    const sizes = genSizes(n);
    const mean = sizes.reduce((a, b) => a + b) / sizes.length;
    const std = Math.sqrt(sizes.reduce((s, v) => s + Math.pow(v - mean, 2), 0) / sizes.length);
    const bg = Math.random() * 0.1 + 0.05;
    const sig = calcSignal(sizes, poly, nr, ex) + bg;
    const noisy = addNoise(sig);
    return { mean, std, sig, noisy, dig: digitize(noisy), bg, snr: sig / 0.005 };
  };

  const generate = async () => {
    setGenerating(true);
    setProgress(0);
    const data = [];
    const polys = ['PE', 'PP', 'PS', 'PET'];
    const counts = [10, 25, 50, 100, 250, 500, 1000];
    const nrs = [0.5, 1.0, 5.0, 10.0];
    const exs = [450, 488, 520];
    let id = 0, total = polys.length * counts.length * nrs.length * exs.length * 3;

    for (const po of polys) {
      for (const ct of counts) {
        for (const nr of nrs) {
          for (const ex of exs) {
            for (let r = 0; r < 3; r++) {
              const m = genMeas(ct, po, nr, ex);
              data.push({
                sample_id: id++, polymer_type: po, particle_count: ct,
                nile_red_conc_ugmL: nr, excitation_nm: ex, replicate: r + 1,
                mean_particle_size_um: m.mean.toFixed(2),
                std_particle_size_um: m.std.toFixed(2),
                total_analog_voltage: m.sig.toFixed(6),
                noisy_analog_voltage: m.noisy.toFixed(6),
                digital_counts: m.dig,
                background_voltage: m.bg.toFixed(6),
                signal_to_noise_ratio: m.snr.toFixed(2)
              });
              setProgress(Math.round((id / total) * 50));
              if (id % 50 === 0) await new Promise(r => setTimeout(r, 0));
            }
          }
        }
      }
    }
    setDataset(data);

    const filt = [];
    let fid = 0;
    const ics = [100, 500, 1000, 5000];
    for (const po of polys) {
      for (const ic of ics) {
        for (let r = 0; r < 5; r++) {
          const bef = genMeas(ic, po, 5, 488);
          const szs = genSizes(ic).filter(s => Math.random() > 0.95 * (1 - Math.exp(-s / 1000)));
          const aft = genMeas(szs.length, po, 5, 488);
          filt.push({
            sample_id: fid++, polymer_type: po,
            initial_particle_count: ic, final_particle_count: szs.length,
            before_digital_counts: bef.dig, after_digital_counts: aft.dig,
            before_analog_voltage: bef.sig.toFixed(6),
            after_analog_voltage: aft.sig.toFixed(6),
            removal_efficiency: ((ic - szs.length) / ic * 100).toFixed(2),
            replicate: r + 1
          });
          setProgress(50 + Math.round((fid / 80) * 50));
        }
      }
    }
    setFiltrationDataset(filt);
    setProgress(100);
    setGenerating(false);
  };

  const toCSV = (d) => {
    if (!d || !d.length) return '';
    const h = Object.keys(d[0]);
    return [h.join(','), ...d.map(r => h.map(k => r[k]).join(','))].join('\n');
  };

  const download = (d, name) => {
    const blob = new Blob([toCSV(d)], { type: 'text/csv' });
    const a = document.createElement('a');
    a.href = URL.createObjectURL(blob);
    a.download = name;
    a.click();
  };

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-4 sm:p-8">
      <div className="max-w-6xl mx-auto">
        <div className="bg-white rounded-lg shadow-xl p-4 sm:p-8">
          <h1 className="text-2xl sm:text-3xl font-bold text-gray-800 mb-2">
            Microplastic Fluorescence Dataset Generator
          </h1>
          <p className="text-gray-600 mb-6 text-sm sm:text-base">
            Generate realistic photodiode response data for Nile Red-stained microplastics
          </p>

          <div className="bg-blue-50 border border-blue-200 rounded-lg p-4 mb-6">
            <p className="font-semibold mb-2 text-blue-900">Dataset Specifications:</p>
            <ul className="text-sm text-blue-800 space-y-1">
              <li>• Main Dataset: 1,344 samples (4 polymers × 7 concentrations × 4 dyes × 3 wavelengths × 3 reps)</li>
              <li>• Filtration Dataset: 80 before/after pairs</li>
              <li>• Features: Realistic noise, size distributions, polymer-specific properties</li>
            </ul>
          </div>

          <button
            onClick={generate}
            disabled={generating}
            className="w-full bg-indigo-600 hover:bg-indigo-700 disabled:bg-gray-400 text-white font-semibold py-4 rounded-lg mb-6 transition-colors"
          >
            {generating ? `Generating... ${progress}%` : '▶ Generate Datasets'}
          </button>

          {generating && (
            <div className="mb-6">
              <div className="w-full bg-gray-200 rounded-full h-3">
                <div className="bg-indigo-600 h-full rounded-full transition-all" style={{ width: `${progress}%` }}></div>
              </div>
            </div>
          )}

          {dataset && filtrationDataset && !generating && (
            <div className="space-y-4">
              <div className="bg-green-50 border border-green-200 rounded-lg p-4">
                <p className="font-semibold text-green-900">✓ Datasets Generated Successfully!</p>
                <p className="text-sm text-green-700">Main: {dataset.length} samples | Filtration: {filtrationDataset.length} samples</p>
              </div>

              <div className="grid sm:grid-cols-2 gap-4">
                <button
                  onClick={() => download(dataset, 'microplastic_main_dataset.csv')}
                  className="bg-green-600 hover:bg-green-700 text-white font-semibold py-3 rounded-lg"
                >
                  ⬇ Download Main Dataset
                </button>
                <button
                  onClick={() => download(filtrationDataset, 'microplastic_filtration_dataset.csv')}
                  className="bg-green-600 hover:bg-green-700 text-white font-semibold py-3 rounded-lg"
                >
                  ⬇ Download Filtration Dataset
                </button>
              </div>

              <div className="mt-6">
                <h3 className="text-lg font-semibold mb-3">Dataset Preview (First 10 rows)</h3>
                <div className="overflow-x-auto border rounded-lg">
                  <table className="min-w-full text-sm">
                    <thead className="bg-gray-50">
                      <tr>
                        {Object.keys(dataset[0]).slice(0, 7).map(k => (
                          <th key={k} className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase">
                            {k.replace(/_/g, ' ')}
                          </th>
                        ))}
                      </tr>
                    </thead>
                    <tbody className="bg-white divide-y">
                      {dataset.slice(0, 10).map((row, i) => (
                        <tr key={i} className="hover:bg-gray-50">
                          {Object.values(row).slice(0, 7).map((v, j) => (
                            <td key={j} className="px-3 py-2 whitespace-nowrap text-gray-700">{v}</td>
                          ))}
                        </tr>
                      ))}
                    </tbody>
                  </table>
                </div>
              </div>
            </div>
          )}
        </div>

        <div className="bg-white rounded-lg shadow-xl p-4 sm:p-8 mt-6">
          <h2 className="text-xl sm:text-2xl font-bold mb-4">How to Use</h2>
          <div className="space-y-3 text-sm sm:text-base text-gray-700">
            <div>
              <h3 className="font-semibold text-lg mb-1">1. Generate & Download</h3>
              <p>Click the button above to create datasets. Download both CSV files.</p>
            </div>
            <div>
              <h3 className="font-semibold text-lg mb-1">2. Analysis Options</h3>
              <ul className="list-disc ml-6 space-y-1">
                <li>Import into Python (pandas), R, or Excel</li>
                <li>Build ML models to classify polymer types</li>
                <li>Analyze filtration efficiency</li>
                <li>Create calibration curves</li>
              </ul>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
};

ReactDOM.render(<App />, document.getElementById('root'));
    </script>
</body>
</html>
