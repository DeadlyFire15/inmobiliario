<script>
	import { onMount } from 'svelte';
	import * as XLSX from 'xlsx';

	let items = [];
	let filteredItems = [];
	let showForm = false;
	let selectedArea = 'Todas';
	let editingId = null;
	let areas = ['Dirección', 'Hospitalización'];
	let showNewAreaInput = false;
	let newAreaName = '';
	let editingAreaIndex = null;
	let editingAreaName = '';
	let filterValues = {};
	let showImportExcel = false;
	let generalAreaForImport = 'Dirección';
	let importFileInput;
	let isImporting = false;
	let importProgress = '';
	let importStats = { total: 0, success: 0, errors: 0, errorList: [] };

	// Formulario
	let formData = {
		numero: '',
		area: 'Dirección',
		areaAdscripcion: '',
		descripcion: '',
		unidadMedida: '',
		claveInventarial: '',
		resguardatario: '',
		marca: '',
		modelo: '',
		numSerie: '',
		estadoFisico: 'Bueno',
		tipoIncidencia: 'Ninguno',
		valorEstimado: '',
		observacion: ''
	};
	let estadoStats = { Bueno: 0, Regular: 0, Malo: 0, 'Reposición': 0, total: 0 };
	const estadosFisicos = ['Bueno', 'Regular', 'Malo', 'Reposición'];
	let incidenciaStats = {
		total: 0,
		adecuados: 0,
		conIncidencias: 0,
		noLocalizados: 0,
		sinResguardo: 0,
		sinEtiquetado: 0,
		bajAdmin: 0,
		bajDictamen: 0,
		transferencia: 0,
		malEstado: 0,
		inservibles: 0,
		mantenimiento: 0,
		siniestro: 0
	};
	const tiposIncidencias = [
		'Ninguno',
		'No localizado o extraviado',
		'Sin resguardo firmado',
		'Sin etiquetado o número de inventario visible',
		'En proceso de baja administrativa',
		'En dictamen técnico para baja',
		'En trámite de transferencia a otra área',
		'En mal estado de uso',
		'Inservible o irreparable',
		'Requiere mantenimiento preventivo o correctivo',
		'Con reporte de siniestro o robo'
	];
	
	// Contadores de categorías especiales
	let specialStats = {
		sinEtiquetado: 0,
		malEstado: 0,
		bajaAdministrativa: 0
	};

	// Función para obtener estadísticas de estados físicos
	function getEstadoStats() {
		const stats = { Bueno: 0, Regular: 0, Malo: 0, 'Reposición': 0, total: 0 };
		items.forEach((item) => {
			const estado = item.estadoFisico || 'Bueno';
			if (stats.hasOwnProperty(estado)) {
				stats[estado]++;
			}
			stats.total++;
		});
		return stats;
	}

	// Función para obtener estadísticas de incidencias
	function getIncidenciaStats() {
		const stats = {
			total: items.length,
			adecuados: 0,
			conIncidencias: 0,
			noLocalizados: 0,
			sinResguardo: 0,
			sinEtiquetado: 0,
			bajAdmin: 0,
			bajDictamen: 0,
			transferencia: 0,
			malEstado: 0,
			inservibles: 0,
			mantenimiento: 0,
			siniestro: 0
		};

		items.forEach((item) => {
			const incidencia = item.tipoIncidencia || 'Ninguno';
			if (incidencia === 'Ninguno') {
				stats.adecuados++;
			} else {
				stats.conIncidencias++;
				if (incidencia === 'No localizado o extraviado') stats.noLocalizados++;
				else if (incidencia === 'Sin resguardo firmado') stats.sinResguardo++;
				else if (incidencia === 'Sin etiquetado o número de inventario visible') stats.sinEtiquetado++;
				else if (incidencia === 'En proceso de baja administrativa') stats.bajAdmin++;
				else if (incidencia === 'En dictamen técnico para baja') stats.bajDictamen++;
				else if (incidencia === 'En trámite de transferencia a otra área') stats.transferencia++;
				else if (incidencia === 'En mal estado de uso') stats.malEstado++;
				else if (incidencia === 'Inservible o irreparable') stats.inservibles++;
				else if (incidencia === 'Requiere mantenimiento preventivo o correctivo') stats.mantenimiento++;
				else if (incidencia === 'Con reporte de siniestro o robo') stats.siniestro++;
			}
		});

		return stats;
	}

	// Función para obtener estadísticas de categorías especiales
	function getSpecialStats() {
		const stats = {
			sinEtiquetado: 0,
			malEstado: 0,
			bajaAdministrativa: 0
		};

		items.forEach((item) => {
			// Contar bienes sin etiquetado o número de inventario visible (N/A o vacío)
			const claveValue = (item.claveInventarial || '').toString().trim().toUpperCase();
			if (claveValue === 'N/A' || claveValue === 'NA' || claveValue === '') {
				stats.sinEtiquetado++;
			}

			// Contar bienes en mal estado de uso
			if (item.estadoFisico === 'Malo') {
				stats.malEstado++;
			}

			// Contar bienes en proceso de baja administrativa
			const observacionValue = (item.observacion || '').toString().toLowerCase();
			if (observacionValue.includes('baja')) {
				stats.bajaAdministrativa++;
			}
		});

		return stats;
	}

	function normalizeEstadoFisico(valor) {
		if (!valor) return 'Bueno';
		const valorLower = valor.toLowerCase().trim();
		if (valorLower === 'b') return 'Bueno';
		if (valorLower === 'r') return 'Regular';
		if (valorLower === 'm') return 'Malo';
		if (valorLower === 'rep') return 'Reposición';
		// Si es un valor conocido, devolverlo como está
		if (['bueno', 'regular', 'malo', 'reposición'].includes(valorLower)) {
			return valor.charAt(0).toUpperCase() + valor.slice(1).toLowerCase();
		}
		return 'Bueno'; // Por defecto
	}

	// Cargar datos del localStorage
	onMount(() => {
		const savedItems = localStorage.getItem('inventoryItems');
		if (savedItems) {
			items = JSON.parse(savedItems);
		}
		const savedAreas = localStorage.getItem('inventoryAreas');
		if (savedAreas) {
			areas = JSON.parse(savedAreas);
		}
		if (areas.length > 0) {
			formData.area = areas[0];
		}
		filterItems();
		estadoStats = getEstadoStats();
		incidenciaStats = getIncidenciaStats();
		specialStats = getSpecialStats();
	});

	// Guardar en localStorage
	function saveToStorage() {
		localStorage.setItem('inventoryItems', JSON.stringify(items));
		localStorage.setItem('inventoryAreas', JSON.stringify(areas));
		filterItems();
	}

	// Agregar nueva área
	function addNewArea() {
		if (!newAreaName.trim()) {
			alert('Por favor ingresa un nombre para el área');
			return;
		}

		if (areas.includes(newAreaName.trim())) {
			alert('Esta área ya existe');
			return;
		}

		areas = [...areas, newAreaName.trim()];
		formData.area = newAreaName.trim();
		newAreaName = '';
		showNewAreaInput = false;
		saveToStorage();
	}

	// Editar área
	function startEditArea(index) {
		editingAreaIndex = index;
		editingAreaName = areas[index];
	}

	// Guardar área editada
	function saveEditArea() {
		if (!editingAreaName.trim()) {
			alert('Por favor ingresa un nombre para el área');
			return;
		}

		if (areas.includes(editingAreaName.trim()) && areas[editingAreaIndex] !== editingAreaName.trim()) {
			alert('Esta área ya existe');
			return;
		}

		const oldName = areas[editingAreaIndex];
		areas[editingAreaIndex] = editingAreaName.trim();

		// Actualizar items que tenían este área
		items = items.map((item) => ({
			...item,
			area: item.area === oldName ? editingAreaName.trim() : item.area
		}));

		editingAreaIndex = null;
		editingAreaName = '';
		saveToStorage();
	}

	// Eliminar área
	function deleteArea(index) {
		const areaName = areas[index];
		const itemsInArea = items.filter((item) => item.area === areaName).length;

		if (itemsInArea > 0) {
			alert(`No se puede eliminar "${areaName}" porque tiene ${itemsInArea} item(s) asignado(s)`);
			return;
		}

		if (!confirm(`¿Estás seguro de que deseas eliminar el área "${areaName}"?`)) {
			return;
		}

		areas = areas.filter((_, i) => i !== index);
		if (formData.area === areaName && areas.length > 0) {
			formData.area = areas[0];
		}
		saveToStorage();
	}

	// Importar datos desde Excel
	function handleExcelImport(event) {
		const file = event.target.files[0];
		if (!file) return;

		isImporting = true;
		importProgress = 'Leyendo archivo...';
		importStats = { total: 0, success: 0, errors: 0, errorList: [] };

		setTimeout(() => {
			const reader = new FileReader();
			reader.onload = (e) => {
				try {
					const data = e.target.result;
					const workbook = XLSX.read(data, { type: 'array' });

					// Procesar cada hoja del archivo
					let allImportedItems = [];
					const sheetNames = workbook.SheetNames;

					sheetNames.forEach((sheetName, sheetIndex) => {
						importProgress = `Procesando hoja ${sheetIndex + 1} de ${sheetNames.length}: "${sheetName}"`;

						const sheet = workbook.Sheets[sheetName];
						const jsonData = XLSX.utils.sheet_to_json(sheet, { defval: '' });

						// Usar el nombre de la hoja como categoría
						const categoryFromSheet = sheetName.trim();

						// Mapear y procesar filas
						jsonData.forEach((row, rowIndex) => {
							try {
								const actualRowNumber = rowIndex + 2;

								// Mapear campos del Excel a nuestro sistema
							const areaAdscripcion = row['Área de Adscripción'] || row['Área de adscripción'] || row['Adscripción'] || row['Adscripcion'] || '';
							const descripcion = row['Descripción'] || row['Descripcion'] || row['descripcion'] || '';
							const unidadMedida = row['Unidad de Medida'] || row['Unidad de medida'] || row['Medida'] || row['medida'] || row['unidad'] || '';
							const claveInventarial = row['Clave inventarial'] || row['Clave Inventarial'] || row['clave'] || '';
							const resguardatario = row['Resguardatario'] || row['resguardatario'] || '';
							const marca = row['Marca'] || row['marca'] || '';
							const modelo = row['Modelo'] || row['modelo'] || '';
							const numSerie = row['Num. de serie'] || row['Num de serie'] || row['serie'] || row['Num. de serie'] || '';
							const estadoFisicoRaw = row['Estado físico'] || row['Estado fisico'] || row['físico'] || row['estado'] || 'Bueno';
							const estadoFisico = normalizeEstadoFisico(estadoFisicoRaw);
							const valorEstimado = row['Valor estimado'] || row['Valor Estimado'] || row['estimado'] || row['valor'] || '';
							const observacion = row['Observación'] || row['Observacion'] || row['observacion'] || '';

// Validar que tenga descripción
							if (!descripcion.trim()) {
								importStats.errors++;
								if (importStats.errorList.length < 10) {
									importStats.errorList.push(
										`Hoja "${sheetName}" fila ${actualRowNumber}: Falta descripción`
										);
									}
									return;
								}

								// Obtener próximo número para la categoría
								const categoryItems = items.concat(allImportedItems).filter((item) => item.area === categoryFromSheet);
								const nextNum = String(categoryItems.length + 1).padStart(2, '0');

								// Crear objeto con estructura correcta
								const newItem = {
									id: Date.now() + Math.random() * 10000,
									numero: nextNum,
									area: categoryFromSheet,
								areaAdscripcion: areaAdscripcion || 'N/A',
								descripcion: descripcion.trim() || 'N/A',
								unidadMedida: unidadMedida || 'N/A',
							claveInventarial: claveInventarial.trim() || 'N/A',
								resguardatario: resguardatario || 'N/A',
								marca: marca || 'N/A',
								modelo: modelo || 'N/A',
								numSerie: numSerie || 'N/A',
									estadoFisico: estadoFisico || 'Bueno',
									valorEstimado: valorEstimado || '',
									observacion: observacion || ''
								};

								allImportedItems.push(newItem);
								importStats.success++;
							} catch (rowError) {
								importStats.errors++;
								if (importStats.errorList.length < 10) {
									importStats.errorList.push(
										`Hoja "${sheetName}" fila ${rowIndex + 2}: ${rowError.message}`
									);
								}
							}
						});
					});

					// Agregar items al array principal
					if (allImportedItems.length > 0) {
						items = [...items, ...allImportedItems];

						// Agregar categorías nuevas si no existen
						allImportedItems.forEach((item) => {
							if (!areas.includes(item.area)) {
								areas = [...areas, item.area];
							}
						});

						saveToStorage();
					}
					estadoStats = getEstadoStats();
					incidenciaStats = getIncidenciaStats();
				specialStats = getSpecialStats();
					let message = `✅ Importación completada:\n${importStats.success} registros guardados`;
					if (importStats.errors > 0) {
						message += `\n⚠️ ${importStats.errors} registros omitidos`;
						if (importStats.errorList.length > 0) {
							message += `\n\nPrimeros errores:\n${importStats.errorList.join('\n')}`;
						}
					}
					alert(message);

					showImportExcel = false;
					if (importFileInput) importFileInput.value = '';
				} catch (error) {
					isImporting = false;
					importProgress = '';
					alert('Error al procesar el archivo Excel:\n' + error.message);
				}
			};

			reader.onerror = () => {
				isImporting = false;
				importProgress = '';
				alert('Error al leer el archivo');
			};

			reader.readAsArrayBuffer(file);
		}, 0);
	}

	// Calcular siguiente número para el área
	function getNextNumber() {
		const areaItems = items.filter((item) => item.area === formData.area);
		if (areaItems.length === 0) {
			return '01';
		}
		const maxNum = Math.max(...areaItems.map((item) => parseInt(item.numero) || 0));
		return String(maxNum + 1).padStart(2, '0');
	}

	// Filtrar items
	function filterItems() {
		let filtered = items;

		// Filtro por área
		if (selectedArea !== 'Todas') {
			filtered = filtered.filter((item) => item.area === selectedArea);
		}

		// Filtros dinámicos por columna
		filtered = filtered.filter((item) => {
			for (const key in filterValues) {
				if (filterValues[key]) {
					const value = String(item[key]).toLowerCase();
					if (!value.includes(filterValues[key].toLowerCase())) {
						return false;
					}
				}
			}
			return true;
		});

		filteredItems = filtered;
	}

	// Agregar o actualizar item
	function handleSubmit() {
		if (!formData.descripcion) {
			alert('Por favor completa la descripción');
			return;
		}

		if (!editingId) {
			formData.numero = getNextNumber();
		}

		if (editingId) {
			const index = items.findIndex((item) => item.id === editingId);
			if (index !== -1) {
				items[index] = { ...formData, id: editingId };
				editingId = null;
			}
		} else {
			items = [
				...items,
				{
					...formData,
					id: Date.now()
				}
			];
		}

		resetForm();
		saveToStorage();
		estadoStats = getEstadoStats();
		incidenciaStats = getIncidenciaStats();
		specialStats = getSpecialStats();
	}

	// Editar item
	function editItem(item) {
		formData = { ...item };
		editingId = item.id;
		showForm = true;
		window.scrollTo({ top: 0, behavior: 'smooth' });
	}

	// Eliminar item
	function deleteItem(id) {
		 if (confirm('¿Estás seguro de que deseas eliminar este item?')) {
			items = items.filter((item) => item.id !== id);
			saveToStorage();
			estadoStats = getEstadoStats();
			incidenciaStats = getIncidenciaStats();
			specialStats = getSpecialStats();
		}
	}

	// Resetear formulario
	function resetForm() {
		formData = {
			numero: getNextNumber(),
			area: areas.length > 0 ? areas[0] : 'Dirección',
			areaAdscripcion: '',
			descripcion: '',
			unidadMedida: '',
			claveInventarial: '',
			resguardatario: '',
			marca: '',
			modelo: '',
			numSerie: '',
			estadoFisico: 'Bueno',
			tipoIncidencia: 'Ninguno',
			valorEstimado: '',
			observacion: ''
		};
		editingId = null;
		showForm = false;
	}

	// Reaccionar a cambios de filtro
	$: if (selectedArea !== undefined || Object.keys(filterValues).length > 0) {
		filterItems();
	}

	// Limpiar datos (para testing)
	function clearAll() {
		if (confirm('¿Estás seguro de que deseas eliminar TODOS los datos?')) {
			items = [];
			localStorage.removeItem('inventoryItems');
			filterItems();
		}
	}

	// Exportar a CSV
	function exportCSV() {
		const headers = [
			'N°',
			'Categoría',
			'Área de Adscripción',
			'Descripción',
			'Unidad',
			'Clave Inventarial',
			'Resguardatario',
			'Marca',
			'Modelo',
			'Serie',
			'Estado',
			'Incidencia',
			'Valor',
			'Observación'
		];

		const rows = filteredItems.map((item) => [
			item.numero,
			item.area,
			item.areaAdscripcion,
			item.descripcion,
			item.unidadMedida,
			item.claveInventarial,
			item.resguardatario,
			item.marca,
			item.modelo,
			item.numSerie,
			item.estadoFisico,
			item.tipoIncidencia || 'Ninguno',
			item.valorEstimado,
			item.observacion
		]);

		const csv = [headers, ...rows].map((row) => row.map((cell) => `"${cell}"`).join(',')).join('\n');

		const blob = new Blob([csv], { type: 'text/csv' });
		const url = window.URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = `inventario_${new Date().toISOString().split('T')[0]}.csv`;
		a.click();
	}
</script>

<div class="min-h-screen bg-gray-50 py-8 px-4 sm:px-6 lg:px-8">
	<div class="mx-auto max-w-7xl">
		<!-- Header -->
		<div class="mb-8 bg-gradient-to-r from-blue-600 to-indigo-600 rounded-lg p-8 text-white shadow-lg">
			<h1 class="text-4xl font-bold">📊 Sistema de Inventario Patrimonial</h1>
			<p class="mt-2 text-lg text-blue-100">Gestión integral de bienes y activos - Sindicatura</p>
		</div>

		<!-- Estadísticas de estados físicos -->
		{#if items.length > 0}
			<div class="mb-8 grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-5">
				<div class="rounded-lg bg-gradient-to-br from-green-50 to-green-100 p-6 border border-green-300 shadow-md hover:shadow-lg transition">
					<div class="text-sm font-semibold text-green-700 uppercase tracking-wide">Total Items</div>
					<div class="mt-3 text-4xl font-bold text-green-900">{estadoStats.total}</div>
					<div class="mt-2 text-xs text-green-600">bienes registrados</div>
				</div>
				<div class="rounded-lg bg-gradient-to-br from-blue-50 to-blue-100 p-6 border border-blue-300 shadow-md hover:shadow-lg transition">
					<div class="text-sm font-semibold text-blue-700 uppercase tracking-wide">✅ Bueno</div>
					<div class="mt-3 text-4xl font-bold text-blue-900">{estadoStats.Bueno}</div>
					<div class="mt-2 text-xs text-blue-600">{estadoStats.total > 0 ? (estadoStats.Bueno / estadoStats.total * 100).toFixed(1) : 0}%</div>
				</div>
				<div class="rounded-lg bg-gradient-to-br from-yellow-50 to-yellow-100 p-6 border border-yellow-300 shadow-md hover:shadow-lg transition">
					<div class="text-sm font-semibold text-yellow-700 uppercase tracking-wide">⚠️ Regular</div>
					<div class="mt-3 text-4xl font-bold text-yellow-900">{estadoStats.Regular}</div>
					<div class="mt-2 text-xs text-yellow-600">{estadoStats.total > 0 ? (estadoStats.Regular / estadoStats.total * 100).toFixed(1) : 0}%</div>
				</div>
				<div class="rounded-lg bg-gradient-to-br from-orange-50 to-orange-100 p-6 border border-orange-300 shadow-md hover:shadow-lg transition">
					<div class="text-sm font-semibold text-orange-700 uppercase tracking-wide">❌ Malo</div>
					<div class="mt-3 text-4xl font-bold text-orange-900">{estadoStats.Malo}</div>
					<div class="mt-2 text-xs text-orange-600">{estadoStats.total > 0 ? (estadoStats.Malo / estadoStats.total * 100).toFixed(1) : 0}%</div>
				</div>
				<div class="rounded-lg bg-gradient-to-br from-red-50 to-red-100 p-6 border border-red-300 shadow-md hover:shadow-lg transition">
					<div class="text-sm font-semibold text-red-700 uppercase tracking-wide">🔄 Reposición</div>
					<div class="mt-3 text-4xl font-bold text-red-900">{estadoStats['Reposición']}</div>
					<div class="mt-2 text-xs text-red-600">{estadoStats.total > 0 ? (estadoStats['Reposición'] / estadoStats.total * 100).toFixed(1) : 0}%</div>
				</div>
			</div>
		{/if}

		<!-- Reporte de Incidencias Patrimoniales -->
		{#if items.length > 0}
			<div class="mb-8 rounded-lg bg-white p-8 shadow-lg border-2 border-blue-200">
				<h2 class="mb-6 text-2xl font-bold text-gray-900">📋 Reporte de Bienes Patrimoniales</h2>
				
				<div class="space-y-6">
					<!-- Resumen General -->
					<div class="rounded-lg bg-blue-50 p-6 border border-blue-200">
						<div class="grid grid-cols-1 gap-6 sm:grid-cols-2">
							<div>
								<p class="text-sm font-medium text-gray-700 mb-2">De un total de <span class="text-2xl font-bold text-blue-900">{incidenciaStats.total}</span> bienes registrados en el inventario patrimonial de esta Unidad Administrativa:</p>
							</div>
							<div>
								<p class="text-sm font-medium text-gray-700 mb-2"><span class="text-2xl font-bold text-green-700">{incidenciaStats.adecuados}</span> bienes se encuentran en condiciones adecuadas de uso.</p>
							</div>
						</div>
					</div>

					<!-- Bienes con Incidencias -->
					<div class="rounded-lg bg-yellow-50 p-6 border border-yellow-200">
						<p class="text-lg font-bold text-gray-900 mb-4"><span class="text-2xl text-yellow-700">{incidenciaStats.conIncidencias}</span> bienes presentan las siguientes incidencias:</p>
						
						<div class="space-y-3 ml-4">
							<div class="flex justify-between items-center p-3 bg-white rounded border border-yellow-100">
								<span class="text-sm font-medium text-gray-700">Bienes no localizados o extraviados:</span>
								<span class="text-lg font-bold text-red-600">{incidenciaStats.noLocalizados}</span>
							</div>
							
							<div class="flex justify-between items-center p-3 bg-white rounded border border-yellow-100">
								<span class="text-sm font-medium text-gray-700">Bienes sin resguardo firmado:</span>
								<span class="text-lg font-bold text-red-600">{incidenciaStats.sinResguardo}</span>
							</div>
							
							<div class="flex justify-between items-center p-3 bg-white rounded border border-yellow-100">
								<span class="text-sm font-medium text-gray-700">Bienes en dictamen técnico para baja:</span>
								<span class="text-lg font-bold text-red-600">{incidenciaStats.bajDictamen}</span>
							</div>
							
							<div class="flex justify-between items-center p-3 bg-white rounded border border-yellow-100">
								<span class="text-sm font-medium text-gray-700">Bienes en trámite de transferencia a otra área:</span>
								<span class="text-lg font-bold text-red-600">{incidenciaStats.transferencia}</span>
							</div>
							
							<div class="flex justify-between items-center p-3 bg-white rounded border border-yellow-100">
								<span class="text-sm font-medium text-gray-700">Bienes en mal estado de uso:</span>
								<span class="text-lg font-bold text-red-600">{incidenciaStats.malEstado}</span>
							</div>
							
							<div class="flex justify-between items-center p-3 bg-white rounded border border-yellow-100">
								<span class="text-sm font-medium text-gray-700">Bienes inservibles o irreparables:</span>
								<span class="text-lg font-bold text-red-600">{incidenciaStats.inservibles}</span>
							</div>
							
							<div class="flex justify-between items-center p-3 bg-white rounded border border-yellow-100">
								<span class="text-sm font-medium text-gray-700">Bienes que requieren mantenimiento preventivo o correctivo:</span>
								<span class="text-lg font-bold text-red-600">{incidenciaStats.mantenimiento}</span>
							</div>
							
							<div class="flex justify-between items-center p-3 bg-white rounded border border-yellow-100">
								<span class="text-sm font-medium text-gray-700">Bienes con reporte de siniestro o robo:</span>
								<span class="text-lg font-bold text-red-600">{incidenciaStats.siniestro}</span>
							</div>
							
							<div class="flex justify-between items-center p-3 bg-white rounded border border-red-100">
								<span class="text-sm font-medium text-gray-700">Bienes sin etiquetado o número de inventario visible (clave vacía):</span>
								<span class="text-lg font-bold text-red-600">{specialStats.sinEtiquetado}</span>
							</div>
							
							<div class="flex justify-between items-center p-3 bg-white rounded border border-red-100">
								<span class="text-sm font-medium text-gray-700">Bienes en mal estado de uso:</span>
								<span class="text-lg font-bold text-red-600">{specialStats.malEstado}</span>
							</div>
							
							<div class="flex justify-between items-center p-3 bg-white rounded border border-red-100">
								<span class="text-sm font-medium text-gray-700">Bienes en proceso de baja administrativa:</span>
								<span class="text-lg font-bold text-red-600">{specialStats.bajaAdministrativa}</span>
							</div>
						</div>
					</div>
				</div>
			</div>
		{/if}

		<!-- Botones de acción -->
		<div class="mb-8 flex flex-wrap gap-3">
			<button
				on:click={() => (showForm = !showForm)}
				class={`rounded-lg px-6 py-2 font-semibold ${
					showForm
						? 'bg-red-600 text-white hover:bg-red-700'
						: 'bg-blue-600 text-white hover:bg-blue-700'
				}`}
			>
				{showForm ? 'Cancelar' : '+ Nuevo Item'}
			</button>
			<button
				on:click={() => (showNewAreaInput = !showNewAreaInput)}
				class={`rounded-lg px-6 py-2 font-semibold ${
					showNewAreaInput
						? 'bg-orange-600 text-white hover:bg-orange-700'
						: 'bg-purple-600 text-white hover:bg-purple-700'
				}`}
			>
				{showNewAreaInput ? 'Cancelar' : '+ Nueva Área'}
			</button>
			<button
				on:click={() => (showImportExcel = !showImportExcel)}
				class={`rounded-lg px-6 py-2 font-semibold ${
					showImportExcel
						? 'bg-red-600 text-white hover:bg-red-700'
						: 'bg-indigo-600 text-white hover:bg-indigo-700'
				}`}
			>
				{showImportExcel ? 'Cancelar' : '📤 Importar Excel'}
			</button>
			<button
				on:click={exportCSV}
				disabled={filteredItems.length === 0}
				class="rounded-lg bg-green-600 px-6 py-2 font-semibold text-white hover:bg-green-700 disabled:bg-gray-400"
			>
				📥 Exportar CSV
			</button>
			<button
				on:click={clearAll}
				disabled={items.length === 0}
				class="rounded-lg bg-gray-600 px-6 py-2 font-semibold text-white hover:bg-gray-700 disabled:bg-gray-400"
			>
				🗑️ Limpiar Todo
			</button>
		</div>

		<!-- Agregar Nueva Área -->
		{#if showNewAreaInput}
			<div class="mb-8 rounded-lg bg-purple-50 p-6 shadow-lg">
				<h2 class="mb-6 text-2xl font-bold text-gray-900">Agregar Nueva Área</h2>

				<div class="space-y-4">
					<div>
						<label class="block text-sm font-medium text-gray-700">Nombre del Área *</label>
						<input
							type="text"
							bind:value={newAreaName}
							placeholder="Ej: Servicios Generales, Finanzas"
							class="mt-1 w-full rounded-lg border border-gray-300 px-4 py-2 focus:border-blue-500 focus:outline-none"
						/>
					</div>

					<div class="flex gap-3 pt-4">
						<button
							type="button"
							on:click={addNewArea}
							class="rounded-lg bg-purple-600 px-6 py-2 font-semibold text-white hover:bg-purple-700"
						>
							✅ Agregar Área
						</button>
						<button
							type="button"
							on:click={() => {
								showNewAreaInput = false;
								newAreaName = '';
							}}
							class="rounded-lg bg-gray-400 px-6 py-2 font-semibold text-white hover:bg-gray-500"
						>
							Cancelar
						</button>
					</div>

					<div class="mt-4 rounded-lg bg-purple-100 p-3">
						<p class="text-sm font-semibold text-purple-900">Áreas existentes:</p>
						<div class="mt-2 flex flex-wrap gap-2">
						{#each areas as area, index (index)}
							<div class="flex items-center gap-2 rounded-full bg-purple-600 px-3 py-1 text-xs font-semibold text-white">
								{#if editingAreaIndex === index}
									<input
										type="text"
										bind:value={editingAreaName}
										class="rounded px-2 py-0.5 text-xs text-gray-900"
										autofocus
									/>
									<button
										on:click={saveEditArea}
										class="ml-1 font-bold hover:underline"
									>
										✓
									</button>
									<button
										on:click={() => (editingAreaIndex = null)}
										class="ml-1 font-bold hover:underline"
									>
										✗
									</button>
								{:else}
									<span>{area}</span>
									<button
										on:click={() => startEditArea(index)}
										class="ml-2 hover:underline"
									>
										✏️
									</button>
									<button
										on:click={() => deleteArea(index)}
										class="ml-1 hover:underline"
									>
										✗
									</button>
								{/if}
							</div>
							{/each}
						</div>
					</div>
				</div>
			</div>
		{/if}

		<!-- Importar Excel -->
		{#if showImportExcel}
			<div class="mb-8 rounded-lg bg-indigo-50 p-6 shadow-lg">
				<h2 class="mb-6 text-2xl font-bold text-gray-900">Importar datos desde Excel (Múltiples hojas)</h2>

				{#if isImporting}
					<div class="rounded-lg bg-blue-100 p-4 text-sm text-blue-900 mb-4">
						<div class="flex items-center gap-2 mb-2">
							<div class="w-6 h-6 border-4 border-blue-300 border-t-blue-900 rounded-full animate-spin"></div>
							<span class="font-semibold">Importando...</span>
						</div>
						<p>{importProgress}</p>
						<p class="mt-2 text-xs">Registros procesados: {importStats.success + importStats.errors}</p>
					</div>
				{/if}

				<div class="rounded-lg bg-blue-100 p-4 text-sm text-blue-900 mb-4">
					<p class="font-semibold mb-2">📋 Instrucciones:</p>
					<ul class="list-disc list-inside space-y-1 text-xs">
						<li><strong>Múltiples hojas:</strong> El sistema importará automáticamente todas las hojas del Excel</li>
						<li><strong>Categorización:</strong> Cada hoja se convierte en una categoría (ej: "Dirección", "Hospitalización")</li>
						<li><strong>Columnas esperadas:</strong> Adscripción, Descripción, Medida, Clave Inventarial, Resguardatario, Marca, Modelo, Num. de serie, físico, estimado, Observación</li>
						<li><strong>Datos requeridos:</strong> Descripción y Clave Inventarial son obligatorios</li>
						<li><strong>Campos vacíos:</strong> Se completarán con "N/A" (No aplica)</li>
						<li><strong>Numeración:</strong> Los números se asignarán automáticamente por categoría</li>
						<li><strong>Rendimiento:</strong> Soporta cientos de registros sin problemas</li>
					</ul>
				</div>

				<div class="space-y-4">
					<div>
						<label class="block text-sm font-medium text-gray-700 mb-2">Selecciona archivo Excel (.xlsx, .xls) *</label>
						<input
							type="file"
							bind:this={importFileInput}
							on:change={handleExcelImport}
							accept=".xlsx,.xls"
							disabled={isImporting}
							class="w-full rounded-lg border border-gray-300 px-4 py-2 disabled:bg-gray-100 disabled:cursor-not-allowed"
						/>
					</div>

					<div class="flex gap-3 pt-4">
						<button
							type="button"
							on:click={() => {
								showImportExcel = false;
								if (importFileInput) importFileInput.value = '';
							}}
							disabled={isImporting}
							class="rounded-lg bg-gray-400 px-6 py-2 font-semibold text-white hover:bg-gray-500 disabled:bg-gray-300 disabled:cursor-not-allowed"
						>
							Cerrar
						</button>
					</div>
				</div>
			</div>
		{/if}
		{#if showForm}
			<div class="mb-8 rounded-lg bg-white p-6 shadow-lg">
				<h2 class="mb-6 text-2xl font-bold text-gray-900">
					{editingId ? 'Editar Item' : 'Agregar Nuevo Item'}
				</h2>

				<form on:submit|preventDefault={handleSubmit} class="space-y-6">
					<!-- Primera fila -->
					<div class="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3">
						<div>
							<label class="block text-sm font-medium text-gray-700">Área de Adscripción *</label>
							<select
								bind:value={formData.area}
								on:change={() => {
									formData.numero = getNextNumber();
								}}
								class="mt-1 w-full rounded-lg border border-gray-300 px-4 py-2 focus:border-blue-500 focus:outline-none"
							>
								{#each areas as area}
									<option value={area}>{area}</option>
								{/each}
							</select>
						</div>

						<div>
						<label class="block text-sm font-medium text-gray-700">Área de Adscripción</label>
						<input
							type="text"
							bind:value={formData.areaAdscripcion}
							placeholder="Ej: Dirección General, Servicios Generales"
							class="mt-1 w-full rounded-lg border border-gray-300 px-4 py-2 focus:border-blue-500 focus:outline-none"
						/>
					</div>

					<div>
							<input
								type="text"
								bind:value={formData.unidadMedida}
								placeholder="Ej: Pz, Equipo"
								class="mt-1 w-full rounded-lg border border-gray-300 px-4 py-2 focus:border-blue-500 focus:outline-none"
							/>
						</div>
					</div>

					<!-- Segunda fila -->
					<div>
						<label class="block text-sm font-medium text-gray-700">Descripción *</label>
						<input
							type="text"
							bind:value={formData.descripcion}
							placeholder="Ej: Mesa Escritorio estructura metálica capo café"
							class="mt-1 w-full rounded-lg border border-gray-300 px-4 py-2 focus:border-blue-500 focus:outline-none"
							required
						/>
					</div>

					<!-- Tercera fila -->
					<div class="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3">
						<div>
							<label class="block text-sm font-medium text-gray-700">Clave Inventarial</label>
							<input
								type="text"
								bind:value={formData.claveInventarial}
								placeholder="Ej: 2024-2027-0060-01"
								class="mt-1 w-full rounded-lg border border-gray-300 px-4 py-2 focus:border-blue-500 focus:outline-none"
							/>
						</div>

						<div>
							<label class="block text-sm font-medium text-gray-700">Resguardatario</label>
							<input
								type="text"
								bind:value={formData.resguardatario}
								placeholder="Ej: J. Allbeidi Peñaloza"
								class="mt-1 w-full rounded-lg border border-gray-300 px-4 py-2 focus:border-blue-500 focus:outline-none"
							/>
						</div>

						<div>
							<label class="block text-sm font-medium text-gray-700">Marca</label>
							<input
								type="text"
								bind:value={formData.marca}
								placeholder="Ej: HP, Dell"
								class="mt-1 w-full rounded-lg border border-gray-300 px-4 py-2 focus:border-blue-500 focus:outline-none"
							/>
						</div>
					</div>

					<!-- Cuarta fila -->
					<div class="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3">
						<div>
							<label class="block text-sm font-medium text-gray-700">Modelo</label>
							<input
								type="text"
								bind:value={formData.modelo}
								placeholder="Ej: EHF120P"
								class="mt-1 w-full rounded-lg border border-gray-300 px-4 py-2 focus:border-blue-500 focus:outline-none"
							/>
						</div>

						<div>
							<label class="block text-sm font-medium text-gray-700">Num. de serie</label>
							<input
								type="text"
								bind:value={formData.numSerie}
								placeholder="Ej: 8CC8341TC7"
								class="mt-1 w-full rounded-lg border border-gray-300 px-4 py-2 focus:border-blue-500 focus:outline-none"
							/>
						</div>

						<div>
							<label class="block text-sm font-medium text-gray-700">Estado Físico</label>
							<select
								bind:value={formData.estadoFisico}
								class="mt-1 w-full rounded-lg border border-gray-300 px-4 py-2 focus:border-blue-500 focus:outline-none"
							>
								{#each estadosFisicos as estado}
									<option value={estado}>{estado}</option>
								{/each}
							</select>
						</div>
					</div>

					<!-- Sexta fila - Tipo de Incidencia -->
					<div>
						<label class="block text-sm font-medium text-gray-700">Tipo de Incidencia (Casos Especiales)</label>
						<select
							bind:value={formData.tipoIncidencia}
							class="mt-1 w-full rounded-lg border border-gray-300 px-4 py-2 focus:border-blue-500 focus:outline-none"
						>
							{#each tiposIncidencias as tipo}
								<option value={tipo}>{tipo}</option>
							{/each}
						</select>
						<p class="mt-1 text-xs text-gray-500">Selecciona si el bien tiene alguna situación especial. De lo contrario, deja "Ninguno".</p>
					</div>

					<!-- Quinta fila -->
					<div class="grid grid-cols-1 gap-6 sm:grid-cols-2">
						<div>
							<label class="block text-sm font-medium text-gray-700">Valor Estimado</label>
							<input
								type="number"
								bind:value={formData.valorEstimado}
								placeholder="Ej: 2500.00"
								step="0.01"
								class="mt-1 w-full rounded-lg border border-gray-300 px-4 py-2 focus:border-blue-500 focus:outline-none"
							/>
						</div>

						<div>
							<label class="block text-sm font-medium text-gray-700">Observación</label>
							<input
								type="text"
								bind:value={formData.observacion}
								placeholder="Ej: Requiere mantenimiento"
								class="mt-1 w-full rounded-lg border border-gray-300 px-4 py-2 focus:border-blue-500 focus:outline-none"
							/>
						</div>
					</div>

					<!-- Botones del formulario -->
					<div class="flex gap-3 pt-4">
						<button
							type="submit"
							class="rounded-lg bg-blue-600 px-6 py-2 font-semibold text-white hover:bg-blue-700"
						>
							{editingId ? '✏️ Actualizar' : '✅ Agregar'}
						</button>
						<button
							type="button"
							on:click={resetForm}
							class="rounded-lg bg-gray-400 px-6 py-2 font-semibold text-white hover:bg-gray-500"
						>
							Limpiar
						</button>
					</div>
				</form>
			</div>
		{/if}

		<!-- Filtros por Área -->
		<div class="mb-8 rounded-lg bg-gradient-to-r from-blue-50 to-indigo-50 p-6 shadow-md border border-blue-200">
			<div class="flex items-center justify-between mb-4">
				<h2 class="text-xl font-bold text-gray-900 flex items-center gap-2">
					🔍 Filtrar Inventario
				</h2>
				<button
					on:click={() => {
						selectedArea = 'Todas';
						filterValues = {};
					}}
					class="text-sm px-3 py-1 rounded-lg bg-blue-600 text-white hover:bg-blue-700 transition"
				>
					Limpiar Filtros
				</button>
			</div>

			<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
				<div>
					<label class="block text-sm font-medium text-gray-700 mb-2">Área Administrativa</label>
					<select
						bind:value={selectedArea}
						class="w-full rounded-lg border border-gray-300 px-4 py-2 focus:border-blue-500 focus:ring-2 focus:ring-blue-200 focus:outline-none transition bg-white"
					>
						<option value="Todas">Todas las Áreas</option>
						{#each areas as area}
							<option value={area}>{area}</option>
						{/each}
					</select>
				</div>

				<div>
					<label class="block text-sm font-medium text-gray-700 mb-2">Descripción</label>
					<input
						type="text"
						bind:value={filterValues.descripcion}
						placeholder="Buscar descripción..."
						class="w-full rounded-lg border border-gray-300 px-4 py-2 focus:border-blue-500 focus:ring-2 focus:ring-blue-200 focus:outline-none transition"
					/>
				</div>

				<div>
					<label class="block text-sm font-medium text-gray-700 mb-2">Clave Inventarial</label>
					<input
						type="text"
						bind:value={filterValues.claveInventarial}
						placeholder="Buscar clave..."
						class="w-full rounded-lg border border-gray-300 px-4 py-2 focus:border-blue-500 focus:ring-2 focus:ring-blue-200 focus:outline-none transition"
					/>
				</div>

				<div>
					<label class="block text-sm font-medium text-gray-700 mb-2">Estado Físico</label>
					<select
						bind:value={filterValues.estadoFisico}
						class="w-full rounded-lg border border-gray-300 px-4 py-2 focus:border-blue-500 focus:ring-2 focus:ring-blue-200 focus:outline-none transition bg-white"
					>
						<option value="">Todos los estados</option>
						<option value="Bueno">Bueno</option>
						<option value="Regular">Regular</option>
						<option value="Malo">Malo</option>
						<option value="Reposición">Reposición</option>
					</select>
				</div>
			</div>

			<div class="mt-4 flex items-center justify-between text-sm text-gray-700 bg-white rounded-lg p-3 border border-blue-100">
				<div>
					<span class="font-semibold text-blue-900">{filteredItems.length}</span> de
					<span class="font-semibold text-blue-900">{items.length}</span> bienes mostrados
				</div>
				{#if filteredItems.length === 0 && items.length > 0}
					<span class="text-orange-600 font-medium">Sin resultados - ajusta los filtros</span>
				{/if}
			</div>
		</div>

		<!-- Tabla de datos -->
		<div class="rounded-lg bg-white shadow-lg border border-gray-200 overflow-hidden">
			{#if filteredItems.length === 0}
				<div class="px-6 py-12 text-center">
					<p class="text-gray-500">No hay items que mostrar. Agrega el primero.</p>
				</div>
			{:else}
				<div class="overflow-x-auto">
					<table class="w-full">
						<thead class="bg-gray-100">
							<tr class="border-b">
								<th class="px-4 py-3 text-left text-sm font-semibold text-gray-900">N°</th>
							<th class="px-4 py-3 text-left text-sm font-semibold text-gray-900">Categoría</th>
							<th class="px-4 py-3 text-left text-sm font-semibold text-gray-900">Área de Adscripción</th>
								<th class="px-4 py-3 text-left text-sm font-semibold text-gray-900">Descripción</th>
								<th class="px-4 py-3 text-left text-sm font-semibold text-gray-900">Unidad</th>
								<th class="px-4 py-3 text-left text-sm font-semibold text-gray-900">Clave</th>
								<th class="px-4 py-3 text-left text-sm font-semibold text-gray-900">Resguardatario</th>
								<th class="px-4 py-3 text-left text-sm font-semibold text-gray-900">Marca</th>
								<th class="px-4 py-3 text-left text-sm font-semibold text-gray-900">Modelo</th>
								<th class="px-4 py-3 text-left text-sm font-semibold text-gray-900">Serie</th>
								<th class="px-4 py-3 text-left text-sm font-semibold text-gray-900">Estado</th>
								<th class="px-4 py-3 text-left text-sm font-semibold text-gray-900">Incidencia</th>
								<th class="px-4 py-3 text-left text-sm font-semibold text-gray-900">Valor</th>
								<th class="px-4 py-3 text-left text-sm font-semibold text-gray-900">Observación</th>
								<th class="px-4 py-3 text-center text-sm font-semibold text-gray-900">Acciones</th>
							</tr>
						</thead>
						<tbody>
							{#each filteredItems as item (item.id)}
								<tr class="border-b hover:bg-blue-50 transition">
									<td class="whitespace-nowrap px-4 py-3 text-sm font-medium text-gray-900">{item.numero}</td>
									<td class="whitespace-nowrap px-4 py-3 text-sm">
										<span class="rounded-full bg-blue-100 px-3 py-1 text-xs font-semibold text-blue-800">
											{item.area}
										</span>
									</td>								<td class="whitespace-nowrap px-4 py-3 text-sm text-gray-900">{item.areaAdscripcion}</td>									<td class="px-4 py-3 text-sm text-gray-900">{item.descripcion}</td>
									<td class="whitespace-nowrap px-4 py-3 text-sm text-gray-900">{item.unidadMedida}</td>
									<td class="whitespace-nowrap px-4 py-3 text-sm text-gray-900">{item.claveInventarial}</td>
									<td class="whitespace-nowrap px-4 py-3 text-sm text-gray-900">
										{item.resguardatario}
									</td>
									<td class="whitespace-nowrap px-4 py-3 text-sm text-gray-900">{item.marca}</td>
									<td class="whitespace-nowrap px-4 py-3 text-sm text-gray-900">{item.modelo}</td>
									<td class="whitespace-nowrap px-4 py-3 text-sm text-gray-900">{item.numSerie}</td>
									<td class="whitespace-nowrap px-4 py-3 text-sm">
										<span
											class={`rounded-full px-3 py-1 text-xs font-semibold ${
												item.estadoFisico === 'Bueno'
													? 'bg-green-100 text-green-800'
													: item.estadoFisico === 'Regular'
														? 'bg-yellow-100 text-yellow-800'
														: item.estadoFisico === 'Malo'
															? 'bg-red-100 text-red-800'
															: 'bg-orange-100 text-orange-800'
											}`}
										>
											{item.estadoFisico}
										</span>
									</td>
									<td class="whitespace-nowrap px-4 py-3 text-sm">
										{#if item.tipoIncidencia && item.tipoIncidencia !== 'Ninguno'}
											<span class="rounded-full bg-red-100 px-3 py-1 text-xs font-semibold text-red-800">
												{item.tipoIncidencia}
											</span>
										{:else}
											<span class="rounded-full bg-green-100 px-3 py-1 text-xs font-semibold text-green-800">
												Ninguno
											</span>
										{/if}
									</td>
									<td class="whitespace-nowrap px-4 py-3 text-sm text-gray-900">${item.valorEstimado}</td>
									<td class="px-4 py-3 text-sm text-gray-900">{item.observacion}</td>
									<td class="whitespace-nowrap px-4 py-3 text-center">
										<button
											on:click={() => editItem(item)}
											class="mr-2 inline-block rounded bg-yellow-500 px-3 py-1 text-sm font-semibold text-white hover:bg-yellow-600"
										>
											✏️
										</button>
										<button
											on:click={() => deleteItem(item.id)}
											class="inline-block rounded bg-red-600 px-3 py-1 text-sm font-semibold text-white hover:bg-red-700"
										>
											🗑️
										</button>
									</td>
								</tr>
							{/each}
						</tbody>
					</table>
				</div>
			{/if}
		</div>
	</div>
</div>

<style>
	:global(body) {
		font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', 'Oxygen', 'Ubuntu',
			'Cantarell', 'Fira Sans', 'Droid Sans', 'Helvetica Neue', sans-serif;
	}
</style>
