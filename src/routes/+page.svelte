<script lang="ts">
	import { onMount, tick } from 'svelte';
	import { Button } from '$lib/components/ui/button';
	import { Input } from '$lib/components/ui/input';
	import { Textarea } from '$lib/components/ui/textarea';
	import { Label } from '$lib/components/ui/label';
	import { Slider } from '$lib/components/ui/slider';
	import * as Select from '$lib/components/ui/select';
	import * as Card from '$lib/components/ui/card';
	import * as Carousel from '$lib/components/ui/carousel';
	import * as Sidebar from '$lib/components/ui/sidebar';
	import { Separator } from '$lib/components/ui/separator';
	import { Loader2, LogOut, User, GraduationCap, Calendar, Settings, Play, BookOpen, Menu } from '@lucide/svelte';

	// State
	let isLoggedIn = false;
	let loading = false;
	let generating = false;
	let streamingLogs: string[] = [];
	let timetables: any[] = [];
	let creditWarning: string | null = null;
	let logsContainer: HTMLElement;

// API 서버 베이스 URL
// - Vercel 등 배포 환경에서는 VITE_API_BASE가 없으면 백엔드 고정 주소로 포인트
// - 로컬 개발 시에는 Vite proxy를 타도록 빈 문자열 유지 (VITE_API_BASE 미설정)
const API_BASE =
	import.meta.env.VITE_API_BASE ||
	(typeof window !== 'undefined' && window.location.hostname.endsWith('vercel.app')
		? 'http://14.36.192.223:8000'
		: '');

	// Login
	let studentId = '';
	let password = '';
	let loginError = '';

	// User Info
	let studentInfo: any = null;

	// Settings
	let year = '';
	let semester = '';
	let targetCredits = [18];
	let unavailableTimes = '';

	const years = [
		{ value: '1', label: '1학년' },
		{ value: '2', label: '2학년' },
		{ value: '3', label: '3학년' },
		{ value: '4', label: '4학년' }
	];

	const semesters = [
		{ value: '2025-1', label: '2025년 1학기' },
		{ value: '2025-2', label: '2025년 2학기' }
	];

	onMount(() => {
		const storedId = sessionStorage.getItem('studentId');
		const storedPw = sessionStorage.getItem('password');
		const storedInfo = sessionStorage.getItem('studentInfo');
		const storedSettings = sessionStorage.getItem('settings');

		if (storedId && storedPw && storedInfo) {
			studentId = storedId;
			password = storedPw;
			studentInfo = JSON.parse(storedInfo);
			isLoggedIn = true;

			if (storedSettings) {
				const settings = JSON.parse(storedSettings);
				year = settings.year || '';
				semester = settings.semester || '';
				targetCredits = [settings.targetCredits || 18];
				unavailableTimes = settings.unavailableTimes || '';
			}
		}
	});

	async function handleLogin() {
		if (!studentId || !password) {
			loginError = '학번과 비밀번호를 입력해주세요.';
			return;
		}

		loading = true;
		loginError = '';

		try {
			const res = await fetch(`${API_BASE}/api/auth/login`, {
				method: 'POST',
				mode: 'cors',
				credentials: 'include',
				referrerPolicy: 'no-referrer',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify({ student_id: studentId, password })
			});

			const data = await res.json();

			if (data.success) {
				studentInfo = data.student_info;
				sessionStorage.setItem('studentId', studentId);
				sessionStorage.setItem('password', password);
				sessionStorage.setItem('studentInfo', JSON.stringify(studentInfo));
				isLoggedIn = true;
			} else {
				loginError = data.message || '로그인에 실패했습니다.';
			}
		} catch (e) {
			loginError = '네트워크 오류가 발생했습니다.';
		} finally {
			loading = false;
		}
	}

	function handleLogout() {
		sessionStorage.clear();
		isLoggedIn = false;
		studentInfo = null;
		studentId = '';
		password = '';
		year = '';
		semester = '';
		targetCredits = [18];
		unavailableTimes = '';
		timetables = [];
		streamingLogs = [];
	}

	function saveSettings() {
		const settings = {
			year,
			semester,
			targetCredits: targetCredits[0],
			unavailableTimes
		};
		sessionStorage.setItem('settings', JSON.stringify(settings));
	}

	async function scrollLogsToBottom() {
		await tick();
		if (logsContainer) {
			logsContainer.scrollTop = logsContainer.scrollHeight;
		}
	}

	async function generateTimetable() {
		if (!year || !semester) {
			alert('학년과 학기를 선택해주세요.');
			return;
		}
		// 모바일/태블릿에서 생성 버튼을 누르면 사이드바 토글 버튼을 찾아 클릭해 닫기
		if (typeof window !== 'undefined') {
			const isNarrow = window.matchMedia('(max-width: 1024px)').matches;
			if (isNarrow) {
				const triggerBtn = document.querySelector<HTMLButtonElement>('[data-slot="sidebar-trigger"]');
				triggerBtn?.click();
			}
		}

		saveSettings();
		generating = true;
		streamingLogs = [];
		timetables = [];
		creditWarning = null;

		try {
			const requestBody = {
				action: 'generate',
				student_id: studentId,
				password: password,
				year: year,
				semester: semester,
				target_credits: targetCredits[0],
				unavailable_times: unavailableTimes || '없음'
			};
			console.log('Request body:', requestBody);
			
			const res = await fetch(`${API_BASE}/api/chat`, {
				method: 'POST',
				mode: 'cors',
				credentials: 'include',
				referrerPolicy: 'no-referrer',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify(requestBody)
			});

			if (!res.ok) {
				const errorText = await res.text();
				console.error('API Error:', res.status, errorText);
				throw new Error(`API 요청 실패: ${res.status}`);
			}

			const reader = res.body?.getReader();
			const decoder = new TextDecoder();

			if (!reader) throw new Error('스트리밍을 지원하지 않습니다.');

			while (true) {
				const { done, value } = await reader.read();
				if (done) break;

				const chunk = decoder.decode(value, { stream: true });
				const lines = chunk.split('\n');

				for (const line of lines) {
					if (line.startsWith('data: ')) {
						try {
							const data = JSON.parse(line.slice(6));

							if (data.type === 'status' || data.type === 'progress') {
								streamingLogs = [...streamingLogs, `✓ ${data.message}`];
								await scrollLogsToBottom();
							} else if (data.type === 'llm_log') {
								streamingLogs = [...streamingLogs, `🤖 ${data.message}`];
								await scrollLogsToBottom();
							} else if (data.type === 'llm_token') {
								// 토큰 단위 스트리밍 - 마지막 줄에 추가
								if (streamingLogs.length === 0 || !streamingLogs[streamingLogs.length - 1].startsWith('💬')) {
									streamingLogs = [...streamingLogs, `💬 ${data.content}`];
								} else {
									streamingLogs[streamingLogs.length - 1] += data.content;
									streamingLogs = [...streamingLogs];
								}
								await scrollLogsToBottom();
							} else if (data.type === 'llm_json') {
								// JSON 전체를 보기 좋게 포매팅하여 출력
								streamingLogs = [...streamingLogs, '📋 AI 추천 결과:'];
								try {
									const parsed = JSON.parse(data.content);
									const pretty = JSON.stringify(parsed, null, 2).split('\n');
									for (const line of pretty) {
										streamingLogs = [...streamingLogs, `   ${line}`];
									}
								} catch (err) {
									// 파싱 실패 시 원문을 그대로 줄바꿈하여 출력
									const rawLines = data.content.split('\n');
									for (const line of rawLines) {
										streamingLogs = [...streamingLogs, `   ${line}`];
									}
								}
								await scrollLogsToBottom();
							} else if (data.type === 'llm_issue') {
								streamingLogs = [...streamingLogs, `⚠️ ${data.message}`];
								await scrollLogsToBottom();
							} else if (data.type === 'llm_fixes') {
								if (data.fixes && data.fixes.length > 0) {
									streamingLogs = [...streamingLogs, `🔧 수정 중: ${data.fixes[0]}`];
								}
								await scrollLogsToBottom();
							} else if (data.type === 'result') {
								timetables = data.timetables || [];
								creditWarning = data.warning || null;
							} else if (data.type === 'error') {
								throw new Error(data.message);
							}
						} catch (e) {
							// JSON 파싱 실패 무시
						}
					}
				}
			}
		} catch (e: any) {
			streamingLogs = [...streamingLogs, `❌ 오류: ${e.message}`];
		} finally {
			generating = false;
		}
	}

	// 시간표 그리드 관련
	const DAYS = ['월', '화', '수', '목', '금'];
	// 13교시까지 표시
	const PERIODS = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13];
	const COLORS = [
		'bg-blue-100 text-blue-800',
		'bg-green-100 text-green-800',
		'bg-purple-100 text-purple-800',
		'bg-orange-100 text-orange-800',
		'bg-pink-100 text-pink-800',
		'bg-teal-100 text-teal-800',
		'bg-indigo-100 text-indigo-800',
		'bg-amber-100 text-amber-800'
	];

	interface TimetableCell {
		course: any;
		colorClass: string;
		isStart: boolean;
		rowSpan: number;
	}

	function parseTimeSlots(timeStr: string): { day: number; period: number }[] {
		if (!timeStr) return [];
		const slots: { day: number; period: number }[] = [];
		const dayMap: Record<string, number> = { 월: 0, 화: 1, 수: 2, 목: 3, 금: 4, 토: 5, 일: 6 };

		const clean = timeStr.replace(/\(/g, ',').replace(/\)/g, '').replace(/\//g, ',');
		const parts = clean.split(',');

		let currentDay: number | null = null;
		for (const part of parts) {
			const trimmed = part.trim();
			if (!trimmed) continue;

			const match = trimmed.match(/^([월화수목금토일])(\d+)/);
			if (match) {
				currentDay = dayMap[match[1]] ?? null;
				const period = parseInt(match[2]);
				if (currentDay !== null && !isNaN(period)) {
					slots.push({ day: currentDay, period });
				}
			} else if (currentDay !== null) {
				const period = parseInt(trimmed);
				if (!isNaN(period)) {
					slots.push({ day: currentDay, period });
				}
			}
		}
		return slots;
	}

	function buildTimetableGrid(courses: any[]): Map<string, TimetableCell> {
		const grid = new Map<string, TimetableCell>();

		courses.forEach((course, idx) => {
			const slots = parseTimeSlots(course.time);
			const colorClass = COLORS[idx % COLORS.length];

			const dayGroups = new Map<number, number[]>();
			slots.forEach(({ day, period }) => {
				if (!dayGroups.has(day)) dayGroups.set(day, []);
				dayGroups.get(day)!.push(period);
			});

			dayGroups.forEach((periods, day) => {
				periods.sort((a, b) => a - b);
				const rowSpan = periods.length;

				periods.forEach((period, i) => {
					const key = `${day}-${period}`;
					grid.set(key, {
						course,
						colorClass,
						isStart: i === 0,
						rowSpan: i === 0 ? rowSpan : 0
					});
				});
			});
		});

		return grid;
	}
</script>

{#if !isLoggedIn}
	<!-- Login Screen -->
	<div class="flex min-h-screen items-center justify-center bg-gray-100 p-4">
		<Card.Root class="w-full max-w-sm shadow-lg">
			<Card.Header class="text-center pb-2">
				<div class="flex justify-center mb-2">
					<div class="w-12 h-12 rounded-full bg-blue-600 flex items-center justify-center">
						<BookOpen class="w-6 h-6 text-white" />
					</div>
				</div>
				<Card.Title class="text-xl">시간표 도우미</Card.Title>
			</Card.Header>
			<Card.Content class="pt-2">
				<form onsubmit={(e) => { e.preventDefault(); handleLogin(); }} class="space-y-3">
					<div class="space-y-1.5">
						<Label for="student-id" class="text-xs">학번</Label>
						<Input 
							id="student-id" 
							type="text" 
							placeholder="학번을 입력하세요" 
							bind:value={studentId} 
							disabled={loading}
							class="h-9"
						/>
					</div>
					<div class="space-y-1.5">
						<Label for="password" class="text-xs">비밀번호</Label>
						<Input 
							id="password" 
							type="password" 
							placeholder="연세포털 비밀번호" 
							bind:value={password} 
							disabled={loading}
							class="h-9"
						/>
					</div>
					{#if loginError}
						<p class="text-center text-xs text-red-500">{loginError}</p>
					{/if}
					<Button type="submit" class="w-full h-9" disabled={loading}>
						{#if loading}
							<Loader2 class="mr-2 h-4 w-4 animate-spin" />
							로그인 중...
						{:else}
							로그인
						{/if}
					</Button>
				</form>
			</Card.Content>
		</Card.Root>
	</div>
{:else}
	<Sidebar.Provider>
		<Sidebar.Root collapsible="offcanvas">
			<Sidebar.Header class="border-b">
				<div class="flex items-center gap-2 px-2 py-1">
					<BookOpen class="w-5 h-5 text-blue-600" />
					<span class="font-semibold">시간표 도우미</span>
				</div>
			</Sidebar.Header>

			<Sidebar.Content>
				<!-- User Info -->
				<Sidebar.Group>
					<Sidebar.GroupContent>
						<div class="px-2 py-3">
							<div class="flex items-center gap-3">
								<div class="w-9 h-9 rounded-full bg-blue-100 flex items-center justify-center flex-shrink-0">
									<User class="w-4 h-4 text-blue-600" />
								</div>
								<div class="min-w-0">
									<p class="font-medium text-sm truncate">
										{studentInfo?.name || studentInfo?.stdntNm || '학생'}
									</p>
									<p class="text-xs text-muted-foreground truncate">{studentId}</p>
								</div>
							</div>
							<div class="mt-3 space-y-1.5 text-xs text-muted-foreground">
								<div class="flex items-center gap-2">
									<GraduationCap class="w-3.5 h-3.5 flex-shrink-0" />
									<span class="truncate">{studentInfo?.department || studentInfo?.deptNm || '-'}</span>
								</div>
								<div class="flex items-center gap-2">
									<Calendar class="w-3.5 h-3.5 flex-shrink-0" />
									<span>{studentInfo?.year || studentInfo?.hy || '-'}학년</span>
								</div>
							</div>
						</div>
					</Sidebar.GroupContent>
				</Sidebar.Group>

				<Sidebar.Separator />

				<!-- Settings -->
				<Sidebar.Group>
					<Sidebar.GroupLabel>
						<Settings class="w-3.5 h-3.5 mr-2" />
						설정
					</Sidebar.GroupLabel>
					<Sidebar.GroupContent>
						<div class="px-2 space-y-4">
							<div class="space-y-1.5">
								<Label class="text-xs">학년</Label>
								<Select.Root type="single" bind:value={year} onValueChange={saveSettings}>
									<Select.Trigger class="w-full h-9 text-sm">
										{year ? years.find((y) => y.value === year)?.label : '선택하세요'}
									</Select.Trigger>
									<Select.Content>
										{#each years as y}
											<Select.Item value={y.value}>{y.label}</Select.Item>
										{/each}
									</Select.Content>
								</Select.Root>
							</div>

							<div class="space-y-1.5">
								<Label class="text-xs">학기</Label>
								<Select.Root type="single" bind:value={semester} onValueChange={saveSettings}>
									<Select.Trigger class="w-full h-9 text-sm">
										{semester ? semesters.find((s) => s.value === semester)?.label : '선택하세요'}
									</Select.Trigger>
									<Select.Content>
										{#each semesters as s}
											<Select.Item value={s.value}>{s.label}</Select.Item>
										{/each}
									</Select.Content>
								</Select.Root>
							</div>

							<div class="space-y-1.5">
								<div class="flex justify-between items-center">
									<Label class="text-xs">목표 학점</Label>
									<span class="text-xs font-semibold text-blue-600">{targetCredits[0]}학점</span>
								</div>
								<Slider
									type="multiple"
									value={targetCredits}
									min={9}
									max={24}
									step={0.5}
									onValueChange={(v: number[]) => { targetCredits = v; saveSettings(); }}
									class="mt-2"
								/>
								<div class="flex justify-between text-[10px] text-muted-foreground mt-1">
									<span>9학점</span>
									<span>24학점</span>
								</div>
							</div>

							<div class="space-y-1.5">
								<Label class="text-xs">희망사항</Label>
								<Textarea
									placeholder="예: 금요일 공강, 수요일 오후 비워줘"
									bind:value={unavailableTimes}
									onchange={saveSettings}
									class="text-sm min-h-[72px] resize-none"
								/>
							</div>
						</div>
					</Sidebar.GroupContent>
				</Sidebar.Group>
			</Sidebar.Content>

			<Sidebar.Footer class="border-t">
				<div class="p-2 space-y-2">
					<Button 
						onclick={generateTimetable} 
						disabled={generating || !year || !semester} 
						class="w-full h-9"
					>
						{#if generating}
							<Loader2 class="mr-2 h-4 w-4 animate-spin" />
							생성 중...
						{:else}
							<Play class="mr-2 h-4 w-4" />
							시간표 생성
						{/if}
					</Button>
					<Button variant="ghost" class="w-full h-8 text-xs text-muted-foreground" onclick={handleLogout}>
						<LogOut class="mr-1.5 h-3.5 w-3.5" />
						로그아웃
					</Button>
				</div>
			</Sidebar.Footer>
		</Sidebar.Root>

		<Sidebar.Inset class="min-w-0 overflow-x-hidden">
			<!-- Mobile Header -->
			<header class="sticky top-0 z-10 flex h-12 shrink-0 items-center gap-2 border-b bg-background px-4 md:hidden">
				<Sidebar.Trigger class="-ml-1">
					<Menu class="h-5 w-5" />
					<span class="sr-only">메뉴 열기</span>
				</Sidebar.Trigger>
				<Separator orientation="vertical" class="h-4" />
				<span class="text-sm font-medium">시간표 도우미</span>
			</header>

			<!-- Main Content -->
			<main class="flex-1 min-w-0 overflow-y-auto overflow-x-hidden p-4 md:p-6 bg-muted/30">
				<div class="max-w-5xl w-full mx-auto space-y-6">
					<!-- Streaming Logs -->
					{#if streamingLogs.length > 0}
						<div class="bg-white rounded-lg shadow-sm border overflow-hidden">
							<div class="px-4 py-3 border-b bg-muted/50">
								<span class="text-sm font-medium">진행 로그</span>
							</div>
							<div
								bind:this={logsContainer}
								class="p-4 font-mono text-xs max-h-72 overflow-y-auto bg-muted/30"
							>
								{#each streamingLogs as log}
									<div class="py-0.5 whitespace-pre leading-relaxed">{log}</div>
								{/each}
							</div>
						</div>
					{/if}

					<!-- Timetable Results -->
					{#if timetables.length > 0}
						<div class="bg-white rounded-lg shadow-sm border overflow-hidden">
							<div class="px-4 py-3 border-b bg-muted/50">
								<span class="text-sm font-medium">추천 시간표</span>
							</div>
							<div class="p-4">
								<Carousel.Root opts={{ align: 'start' }} class="w-full">
									<Carousel.Content class="-ml-4">
										{#each timetables.slice(0, 10) as timetable, idx}
											{@const grid = buildTimetableGrid(timetable.courses)}
											<Carousel.Item class="pl-4 basis-full lg:basis-1/2">
												<Card.Root class="h-full">
													<Card.Header class="pb-3">
														<div class="flex items-center justify-between">
															<Card.Title class="text-base">시간표 {idx + 1}</Card.Title>
															<div class="flex items-center gap-1.5">
																<span class="bg-blue-100 text-blue-700 px-2 py-0.5 rounded-md text-xs font-medium">
																	{timetable.total_credits}학점
																</span>
																<span class="bg-green-100 text-green-700 px-2 py-0.5 rounded-md text-xs font-medium">
																	공강 {timetable.free_days}일
																</span>
															</div>
														</div>
													</Card.Header>
													<Card.Content class="pt-0">
														<!-- Grid -->
														<div
															class="border rounded-md overflow-hidden grid text-[11px]"
															style="grid-template-columns: 32px repeat(5, 1fr); grid-template-rows: 28px repeat(13, minmax(36px, auto));"
														>
															<!-- Header Row -->
															<div class="bg-muted flex items-center justify-center border-r border-b text-muted-foreground font-medium"></div>
															{#each DAYS as day}
																<div class="bg-muted flex items-center justify-center border-r last:border-r-0 border-b font-medium">{day}</div>
															{/each}

															<!-- Period Rows -->
															{#each PERIODS as period}
																<div
																	class="bg-muted/40 flex items-center justify-center border-r border-b text-muted-foreground font-medium"
																	style="grid-column: 1; grid-row: {period + 1};"
																>
																	{period}
																</div>
																{#each DAYS as _, dayIdx}
																	{@const cell = grid.get(`${dayIdx}-${period}`)}
																	{#if cell && cell.isStart}
																		<div
																			class="p-1 {cell.colorClass} border-r last:border-r-0 border-b flex flex-col justify-center items-center overflow-hidden"
																			style="grid-column: {dayIdx + 2}; grid-row: {period + 1} / span {cell.rowSpan};"
																		>
																			<span class="font-semibold text-center leading-tight line-clamp-2">{cell.course.name}</span>
																			{#if cell.course.professor}
																				<span class="text-[9px] opacity-80 text-center mt-0.5">
																					{cell.course.professor}
																				</span>
																			{/if}
																			{#if cell.course.room}
																				<span class="text-[9px] opacity-70 text-center">
																					{cell.course.room}
																				</span>
																			{/if}
																		</div>
																	{:else if !cell}
																		<div
																			class="bg-background border-r last:border-r-0 border-b"
																			style="grid-column: {dayIdx + 2}; grid-row: {period + 1};"
																		></div>
																	{/if}
																{/each}
															{/each}
														</div>
													</Card.Content>
												</Card.Root>
											</Carousel.Item>
										{/each}
									</Carousel.Content>
									{#if timetables.length > 2}
										<Carousel.Previous class="-left-3 h-8 w-8" />
										<Carousel.Next class="-right-3 h-8 w-8" />
									{/if}
								</Carousel.Root>
								{#if creditWarning}
									<div class="mt-4 p-3 bg-amber-50 border border-amber-200 rounded-lg text-amber-800 text-sm">
										{creditWarning}
									</div>
								{/if}
							</div>
						</div>
					{:else if !generating && streamingLogs.length === 0}
						<div class="text-center py-20">
							<div class="w-16 h-16 rounded-full bg-muted flex items-center justify-center mx-auto mb-4">
								<Calendar class="w-8 h-8 text-muted-foreground" />
							</div>
							<h3 class="text-lg font-medium mb-2">시간표를 생성해보세요</h3>
							<p class="text-sm text-muted-foreground">
								<span class="hidden md:inline">왼쪽 설정에서 학년과 학기를 선택한 후</span>
								<span class="md:hidden">메뉴에서 설정을 완료한 후</span><br />
								시간표 생성 버튼을 클릭하세요.
							</p>
						</div>
					{/if}
				</div>
			</main>
		</Sidebar.Inset>
	</Sidebar.Provider>
{/if}
