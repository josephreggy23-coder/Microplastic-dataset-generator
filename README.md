# Microplastic-dataset-generator
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Microplastic Fluorescence Dataset Generator</title>
    <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState } = React;

        const MicroplasticDatasetGenerator = () => {
            const [generating, setGenerating] = useState(false);
            const [dataset, setDataset] = useState(null);
            const [filtrationDataset, setFiltrationDataset] = useState(null);
            const [progress, setProgress] = useState(0);

            const polymerProperties = {
                PE: { quantumYield: 0.38, bindingEfficiency: 0.85, baselineFluorescence: 1200, sizeCoefficient: 1.0 },
                PP: { quantumYield: 0.35, bindingEfficiency: 0.80, baselineFluorescence: 1100, sizeCoefficient: 0.95 },
                PS: { quantumYield: 0.42, bindingEfficiency: 0.90, baselineFluorescence: 1400, sizeCoefficient: 1.05 },
                PET: { quantumYield: 0.33, bindingEfficiency: 0.75, baselineFluorescence: 1000, sizeCoefficient: 0.90 }
            };

            const responsivity = 0.4;
            const darkCurrent = 2e-9;
            const transimpedanceGain = 1e6;
            const adcBits = 12;
            const adcVoltageRange = 5.0;

            const generateParticleSizes = (n) => {
                const sizes = [];
                const mu = Math.log(500);
                const sigma = 0.6;
                
                for (let i = 0; i < n; i++) {
                    let u1 = Math.random();
                    let u2 = Math.random();
                    let z = Math.sqrt(-2 * Math.log(u1)) * Math.cos(2 * Math.PI * u2);
                    let size = Math.exp(mu + sigma * z);
                    size = Math.max(5, Math.min(5000, size));
                    sizes.push(size);
                }
                return sizes;
            };

            const calculateFluorescence = (sizes, polymerType, nileRedConc, excitationWl) => {
                const props = polymerProperties[polymerType];
                const concFactor = nileRedConc / (nileRedConc + 2.0);
                const wlFactor = Math.exp(-Math.pow(excitationWl - 550, 2) / (2 * 40 * 40));
                
                let totalSignal = 0;
                sizes.forEach(size => {
                    const relativeIntensity = Math.pow(size / 500, 2) * (0.85 + Math.random() * 0.3);
                    const photocurrent = props.baselineFluorescence * props.quantumYield * 
                                        props.bindingEfficiency * relativeIntensity * 
                                        concFactor * wlFactor * responsivity * 1e-9;
                    const voltage = photocurrent * transimpedanceGain;
                    totalSignal += voltage;
                });
                
                return totalSignal;
            };

            const addNoise = (signal) => {
                const shotNoise = (Math.random() - 0.5) * 0.002 * Math.sqrt(Math.abs(signal));
                const darkNoise = (Math.random() - 0.5) * darkCurrent * transimpedanceGain * 0.2;
                const johnsonNoise = (Math.random() - 0.5) * 0.002;
                const flickerNoise = (Math.random() - 0.5) * 0.001;
                return signal + shotNoise + darkNoise + johnsonNoise + flickerNoise;
            };

            const digitizeSignal = (analogVoltage) => {
                const clipped = Math.max(0, Math.min(adcVoltageRange, analogVoltage));
                const maxCount = Math.pow(2, adcBits) - 1;
                return Math.round((clipped / adcVoltageRange) * maxCount);
            };

            const generateMeasurement = (nParticles, polymerType, nileRedConc, excitationWl) => {
                const sizes = generateParticleSizes(nParticles);
                const meanSize = sizes.reduce((a, b) => a + b, 0) / sizes.length;
                const stdSize = Math.sqrt(sizes.reduce((sq, n) => sq + Math.pow(n - meanSize, 2), 0) / sizes.length);
                
                const background = Math.random() * 0.1 + 0.05;
                const totalSignal = calculateFluorescence(sizes, polymerType, nileRedConc, excitationWl);
                const totalAnalog = totalSignal + background;
                const noisySignal = addNoise(totalAnalog);
                const digitalCounts = digitizeSignal(noisySignal);
                
                return {
                    meanSize, stdSize, totalAnalog, noisySignal, digitalCounts, background,
                    signalToNoise: totalAnalog / 0.005
                };
            };

            const generateDataset = async () => {
                setGenerating(true);
                setProgress(0);
                
                const mainData = [];
                const polymers = ['PE', 'PP', 'PS', 'PET'];
                const particleCounts = [10, 25, 50, 100, 250, 500, 1000];
                const nileRedConcs = [0.5, 1.0, 5.0, 10.0];
                const excitationWls = [450, 488, 520];
                const replicates = 3;
                
                let sampleId = 0;
                const totalSamples = polymers.length * particleCounts.length * nileRedConcs.length * 
                                    excitationWls.length * replicates;
                
                for (const polymer of polymers) {
                    for (const particleCount of particleCounts) {
                        for (const nrConc of nileRedConcs) {
                            for (const exWl of excitationWls) {
                                for (let rep = 0; rep < replicates; rep++) {
                                    const measurement = generateMeasurement(particleCount, polymer, nrConc, exWl);
                                    
                                    mainData.push({
                                        sample_id: sampleId++,
                                        polymer_type: polymer,
                                        particle_count: particleCount,
                                        nile_red_conc_ugmL: nrConc,
                                        excitation_nm: exWl,
                                        replicate: rep + 1,
                                        mean_particle_size_um: measurement.meanSize.toFixed(2),
                                        std_particle_size_um: measurement.stdSize.toFixed(2),
                                        total_analog_voltage: measurement.totalAnalog.toFixed(6),
                                        noisy_analog_voltage: measurement.noisySignal.toFixed(6),
                                        digital_counts: measurement.digitalCounts,
                                        background_voltage: measurement.background.toFixed(6),
                                        signal_to_noise_ratio: measurement.signalToNoise.toFixed(2)
                                    });
                                    
                                    setProgress(Math.round((sampleId / totalSamples) * 50));
                                    
                                    if (sampleId % 50 === 0) {
                                        await new Promise(resolve => setTimeout(resolve, 0));
                                    }
                                }
                            }
                        }
                    }
                }
                
                setDataset(mainData);
                
                const filtrationData = [];
                let filtrationId = 0;
                const initialCounts = [100, 500, 1000, 5000];
                const filterEfficiency = 0.95;
                
                for (const polymer of polymers) {
                    for (const initialCount of initialCounts) {
                        for (let rep = 0; rep < 5; rep++) {
                            const before = generateMeasurement(initialCount, polymer, 5.0, 488);
                            const beforeSizes = generateParticleSizes(initialCount);
                            const afterSizes = beforeSizes.filter(size => {
                                const retentionProb = filterEfficiency * (1 - Math.exp(-size / 1000));
                                return Math.random() > retentionProb;
                            });
                            
                            const afterCount = afterSizes.length;
                            const after = generateMeasurement(afterCount, polymer, 5.0, 488);
                            const removalEff = (initialCount - afterCount) / initialCount;
                            
                            filtrationData.push({
                                sample_id: filtrationId++,
                                polymer_type: polymer,
                                initial_particle_count: initialCount,
                                final_particle_count: afterCount,
                                before_digital_counts: before.digitalCounts,
                                after_digital_counts: after.digitalCounts,
                                before_analog_voltage: before.totalAnalog.toFixed(6),
                                after_analog_voltage: after.totalAnalog.toFixed(6),
                                removal_efficiency: (removalEff * 100).toFixed(2),
                                replicate: rep + 1
                            });
                            
                            setProgress(50 + Math.round((filtrationId / (polymers.length * initialCounts.length * 5)) * 50));
                        }
                    }
                }
                
                setFiltrationDataset(filtrationData);
                setProgress(100);
                setGenerating(false);
            };

            const convertToCSV = (data) => {
                if (!data || data.length === 0) return '';
                const headers = Object.keys(data[0]);
                const csvRows = [
                    headers.join(','),
                    ...data.map(row => 
                        headers.map(header => {
                            const value = row[header];
                            return typeof value === 'string' && value.includes(',') ? `"${value}"` : value;
                        }).join(',')
                    )
                ];
                return csvRows.join('\n');
            };

            const downloadCSV = (data, filename) => {
                const csv = convertToCSV(data);
                const blob = new Blob([csv], { type: 'text/csv' });
                const url = window.URL.createObjectURL(blob);
                const a = document.createElement('a');
                a.href = url;
                a.download = filename;
                a.click();
                window.URL.revokeObjectURL(url);
            };

            return (
                <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-8">
                    <div className="max-w-6xl mx-auto">
                        <div className="bg-white rounded-lg shadow-xl p-8">
                            <h1 className="text-3xl font-bold text-gray-800 mb-2">
                                Microplastic Fluorescence Dataset Generator
                            </h1>
                            <p className="text-gray-600 mb-6">
                                Generate realistic synthetic photodiode response data for Nile Red-stained microplastics
                            </p>

                            <div className="bg-blue-50 border border-blue-200 rounded-lg p-4 mb-6">
                                <div className="flex items-start gap-3">
                                    <svg className="text-blue-600 mt-0.5 flex-shrink-0" width="20" height="20" fill="currentColor" viewBox="0 0 24 24">
                                        <path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 15h-2v-2h2v2zm0-4h-2V7h2v6z"/>
                                    </svg>
                                    <div className="text-sm text-blue-900">
                                        <p className="font-semibold mb-1">Dataset Specifications:</p>
                                        <ul className="list-disc list-inside space-y-1 ml-2">
                                            <li><strong>Main Dataset:</strong> 1,344 samples</li>
                                            <li><strong>Filtration Dataset:</strong> 80 before/after pairs</li>
                                            <li><strong>Features:</strong> Realistic noise models, size distributions, polymer-specific properties</li>
                                        </ul>
                                    </div>
                                </div>
                            </div>

                            <button
                                onClick={generateDataset}
                                disabled={generating}
                                className="w-full bg-indigo-600 hover:bg-indigo-700 disabled:bg-gray-400 text-white font-semibold py-4 rounded-lg transition-colors flex items-center justify-center gap-2 mb-6"
                            >
                                {generating ? (
                                    <>
                                        <div className="animate-spin rounded-full h-5 w-5 border-b-2 border-white"></div>
                                        Generating... {progress}%
                                    </>
                                ) : (
                                    <>
                                        <svg width="20" height="20" fill="currentColor" viewBox="0 0 24 24">
                                            <path d="M8 5v14l11-7z"/>
                                        </svg>
                                        Generate Datasets
                                    </>
                                )}
                            </button>

                            {generating && (
                                <div className="mb-6">
                                    <div className="w-full bg-gray-200 rounded-full h-3 overflow-hidden">
                                        <div 
                                            className="bg-indigo-600 h-full transition-all duration-300 rounded-full"
                                            style={{ width: `${progress}%` }}
                                        ></div>
                                    </div>
                                </div>
                            )}

                            {dataset && filtrationDataset && !generating && (
                                <div className="space-y-4">
                                    <div className="bg-green-50 border border-green-200 rounded-lg p-4">
                                        <div className="flex items-center gap-3">
                                            <svg className="text-green-600" width="24" height="24" fill="currentColor" viewBox="0 0 24 24">
                                                <path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm-2 15l-5-5 1.41-1.41L10 14.17l7.59-7.59L19 8l-9 9z"/>
                                            </svg>
                                            <div>
                                                <p className="font-semibold text-green-900">Datasets Generated Successfully!</p>
                                                <p className="text-sm text-green-700">
                                                    Main dataset: {dataset.length} samples | Filtration dataset: {filtrationDataset.length} samples
                                                </p>
                                            </div>
                                        </div>
                                    </div>

                                    <div className="grid md:grid-cols-2 gap-4">
                                        <button
                                            onClick={() => downloadCSV(dataset, 'microplastic_main_dataset.csv')}
                                            className="bg-green-600 hover:bg-green-700 text-white font-semibold py-3 rounded-lg transition-colors flex items-center justify-center gap-2"
                                        >
                                            <svg width="20" height="20" fill="currentColor" viewBox="0 0 24 24">
                                                <path d="M19 9h-4V3H9v6H5l7 7 7-7zM5 18v2h14v-2H5z"/>
                                            </svg>
                                            Download Main Dataset
                                        </button>
                                        
                                        <button
                                            onClick={() => downloadCSV(filtrationDataset, 'microplastic_filtration_dataset.csv')}
                                            className="bg-green-600 hover:bg-green-700 text-white font-semibold py-3 rounded-lg transition-colors flex items-center justify-center gap-2"
                                        >
                                            <svg width="20" height="20" fill="currentColor" viewBox="0 0 24 24">
                                                <path d="M19 9h-4V3H9v6H5l7 7 7-7zM5 18v2h14v-2H5z"/>
                                            </svg>
                                            Download Filtration Dataset
                                        </button>
                                    </div>

                                    <div className="mt-6">
                                        <h3 className="text-lg font-semibold text-gray-800 mb-3">Main Dataset Preview (First 10 rows)</h3>
                                        <div className="overflow-x-auto border border-gray-200 rounded-lg">
                                            <table className="min-w-full divide-y divide-gray-200 text-sm">
                                                <thead className="bg-gray-50">
                                                    <tr>
                                                        {Object.keys(dataset[0]).slice(0, 8).map(key => (
                                                            <th key={key} className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                                                                {key.replace(/_/g, ' ')}
                                                            </th>
                                                        ))}
                                                    </tr>
                                                </thead>
                                                <tbody className="bg-white divide-y divide-gray-200">
                                                    {dataset.slice(0, 10).map((row, idx) => (
                                                        <tr key={idx} className="hover:bg-gray-50">
                                                            {Object.values(row).slice(0, 8).map((val, i) => (
                                                                <td key={i} className="px-3 py-2 whitespace-nowrap text-gray-700">
                                                                    {val}
                                                                </td>
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

                        <div className="bg-white rounded-lg shadow-xl p-8 mt-6">
                            <h2 className="text-2xl font-bold text-gray-800 mb-4">How to Use This Dataset</h2>
                            <div className="space-y-4 text-gray-700">
                                <div>
                                    <h3 className="font-semibold text-lg mb-2">1. Generate & Download</h3>
                                    <p>Click "Generate Datasets" to create realistic synthetic data. Download both CSV files.</p>
                                </div>
                                <div>
                                    <h3 className="font-semibold text-lg mb-2">2. Analysis Options</h3>
                                    <ul className="list-disc list-inside ml-4 space-y-1">
                                        <li>Import into Python (pandas), R, or Excel for analysis</li>
                                        <li>Build machine learning models to classify polymer types</li>
                                        <li>Analyze filtration efficiency across different conditions</li>
                                        <li>Create visualizations showing concentration vs. signal relationships</li>
                                    </ul>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            );
        };

        ReactDOM.render(<MicroplasticDatasetGenerator />, document.getElementById('root'));
    </script>
</body>
</html>
