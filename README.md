# prosps.ru: интерактивная витрина, которая продаёт сложные проекты адапитированная под мобильные устройства 

<img src="./prospsgif.gif" alt="site gif"  />

> Продуктовый лендинг с живым UI: фуллскрин-хиро на Swiper 11, кастомные SVG-карты и интерактивные схемы изделий. Архитектура на Next.js + TypeScript + Zustand + Tailwind, анимации на Framer Motion. Ведём пользователя от первого экрана до заявки без трений — и сразу шлём лиды в CRM.

---

## Фуллскрин-хиро со Swiper 11

Фуллскрин-хиро подстраивается под текущий маршрут. Я инициирую `fade`-переходы, динамические буллеты и ручной автоплей, поэтому слайдер рассказывает о направлениях бизнеса, а CTA «Подробнее» ведёт на нужную страницу.

```tsx
const TopSwiper = () => {
  const path = usePathname();
  const section = useMemo(() => navLinks.find((link) => link.href === path), [path]);
  if (path !== '/') return section ? <TopSectionSwiper section={section} /> : null;

  return (
    <Swiper
      effect="fade"
      autoplay={{ delay: 3000, disableOnInteraction: false }}
      speed={1000}
      pagination={{ dynamicBullets: true, clickable: true }}
      loop
      onSwiper={(swiper) => swiper.autoplay?.start?.()}
    >
      {navLinks.slice(1, -1).map((link, index) => (
        <SwiperSlide key={link.label}>
          <TopSwiperSlideContent
            startPage={index === 0}
            href={link.href}
            headerTitle="Мы производим"
            headerTitleItem={link.swiperLabel ?? link.label}
            headerText={mainSwiperText}
            imgSrc={link.mainImgSrc}
          />
        </SwiperSlide>
      ))}
    </Swiper>
  );
};
```

Секции под героем усиливают доверие: **О компании** с фотокаруселью, сетка преимуществ, бегущие ленты клиентов и проектов. Всё построено на кастомных анимациях **Tailwind + Framer Motion**, чтобы интерфейс был живым, но лёгким.

---

## Карты и интерактивные схемы

### Главная карта баз

Главная карта баз — кастомная SVG-сцена. Я всттроил фон в `<svg>`, расположил маркеры по процентным координатам, подсветил активную точку и показал карточку, чья позиция вычисляется по DOM-координатам маркера. Панель баз и карта работают синхронно.

```tsx
const MainMap = () => {
  const [activeId, setActiveId] = useState<number | null>(1);
  const containerRef = useRef<HTMLDivElement>(null);
  const markerRefs = useRef<Record<string, SVGGElement | null>>({});
  const cardPos = useFloatingCardPosition(activeId, containerRef, markerRefs, useWindowWidth());
  const activeBase = useMemo(() => BASES.find((base) => base.id === activeId), [activeId]);

  return (
    <section ref={containerRef} className="relative h-[720px] rounded-xl bg-white">
      <TopPanel />
      <GradientDiv />
      {activeBase && (
        <FloatingCard cardPos={cardPos} containerRef={containerRef} activeItem={activeBase}>
          <FloatingCardContent activeBase={activeBase} />
        </FloatingCard>
      )}
      <ImageWithMarkers
        imgfile={mapImage}
        markerData={BASES}
        activeId={activeId}
        setActiveId={setActiveId}
        markerRefs={markerRefs}
      />
      <BasesPanel data={BASES} activeId={activeId} setActiveId={setActiveId} title="Производственные базы" />
    </section>
  );
};
```

### ItemMap на страницах изделий

Тот же движок используется для **ItemMap** на страницах вагон-домов и КТС. Фотография конструкции встроена в `<svg>`, поверх — маркеры. Автоплей перебирает узлы, пока пользователь не вмешается.

```tsx
const ItemMap = ({ imgFile, markerData }: { imgFile: StaticImageData; markerData: MarkerData[] }) => {
  const [activeId, setActiveId] = useState<number | null>(null);
  const [autoplayOn, setAutoplayOn] = useState(true);
  const containerRef = useRef<HTMLDivElement>(null);
  const markerRefs = useRef<Record<string, SVGGElement | null>>({});
  const cardPos = useFloatingCardPosition(activeId, containerRef, markerRefs, useWindowWidth());
  const activeItem = useMemo(() => markerData.find((marker) => marker.id === activeId), [markerData, activeId]);

  useEffect(() => {
    if (!autoplayOn) return;
    const interval = setInterval(() => {
      setActiveId((prev) => (prev ? ((prev % markerData.length) + 1) : 1));
    }, 4000);
    return () => clearInterval(interval);
  }, [markerData.length, autoplayOn]);

  return (
    <div ref={containerRef} className="relative w-full">
      {activeItem && (
        <FloatingCard cardPos={cardPos} containerRef={containerRef} activeItem={activeItem}>
          <ItemCardContent item={activeItem} />
        </FloatingCard>
      )}
      <ImageWithMarkers
        imgfile={imgFile}
        markerData={markerData}
        activeId={activeId}
        setActiveId={(id) => {
          setAutoplayOn(false);
          setActiveId(id);
        }}
        markerRefs={markerRefs}
      />
    </div>
  );
};
```

### Внутренности `ImageWithMarkers`

```tsx
export const ImageWithMarkers = ({
  activeId,
  setActiveId,
  markerRefs,
  imgfile,
  markerData,
  markerScale,
  className,
  setAutolayOn,
}: ImageWithMarkersProps) => {
  return (
    <svg
      viewBox={`0 0 ${imgfile.width} ${imgfile.height}`}
      className={cn('h-full w-full', className)}
      preserveAspectRatio="xMidYMid meet"
    >
      <image
        className="drop-shadow-2xl"
        href={imgfile.src}
        width={imgfile.width}
        height={imgfile.height}
        onClick={() => setActiveId(null)}
      />
      {[...markerData].reverse().map((marker) => {
        const cx = (marker.x / 100) * imgfile.width;
        const cy = (marker.y / 100) * imgfile.height;
        const scale = (markerScale ?? 100) / 100;

        return (
          <g
            key={marker.id}
            ref={(el) => {
              markerRefs.current[marker.id] = el;
            }}
            transform={`translate(${cx}, ${cy}) scale(${scale})`}
            style={{ cursor: 'pointer' }}
            onMouseEnter={() => {
              setActiveId(marker.id);
              setAutolayOn?.(false);
            }}
            onClick={() => {
              setActiveId(marker.id);
              setAutolayOn?.(false);
            }}
          >
            <MarkerSVG active={activeId === marker.id} />
          </g>
        );
      })}
    </svg>
  );
};
```

**Как это работает:**

- Изображение изделия/карты встроено в `<image>` внутри `<svg>`, что сохраняет пропорции и позволяет рассчитывать координаты.
- Маркеры переводят процентные значения в пиксели и используют `transform`, чтобы точка попадала в нужное место.
- Каждая `<g>` записывает DOM-узел в `markerRefs`, а `useFloatingCardPosition` вычисляет позицию всплывашки через `getBoundingClientRect()`.
- `MarkerSVG` — два `motion.circle` с анимациями: активная точка «дышит», увеличиваясь и меняя цвет.
- `setAutolayOn?.(false)` отключает автоплей при первом взаимодействии, поэтому пользователь может изучать выбранный узел.

---

## Формы и калькулятор

Формы валидируют поля, чистят телефон/почту и отправляют данные в CRM. Я использовал Zustand, чтобы подставить выбранный товар автоматически, и показал одно из двух сообщений: успех или корректную ошибку.

```ts
const onSubmit = async (event: FormEvent<HTMLFormElement>) => {
  event.preventDefault();
  const nextErrors = validateAll(formValues);
  if (Object.keys(nextErrors).length) return setErrors(nextErrors);

  setLoading(true);
  setErrors({});
  setMsg(null);

  try {
    const response = await fetch(apiEndpoint, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        ...formValues,
        cartItems: [formValues.product],
        site: 'Заказ с сайта prosps.ru',
      }),
    });
    const data = await response.json();

    if (data.result === 0) {
      setMsg(data.desc);
    } else {
      setFormValues(initialState);
      setSelectedProductName(null);
      setMsg('Спасибо! Заявка отправлена. Мы свяжемся с вами в течение 24 часов.');
    }
  } catch {
    setMsg('Не удалось отправить. Попробуйте ещё раз.');
  } finally {
    setLoading(false);
  }
};
```

Калькулятор каркасно-тентовых сооружений добавляет параметры ангара, готовя детализированное сообщение менеджеру.

---

## Контакты и социальные каналы

Страница **Контакты** соединяет панель точек с Яндекс-картой: выбор элемента смещает центр и масштаб, маркер меняет иконку и размер. Ниже — карточки менеджеров и анимированные кнопки подписки.

```tsx
const handleActiveIdChange = (id: number | null, geo?: number[]) => {
  setActiveId(id);
  if (!geo) return;
  setCenter([geo[0] - (isMobile ? 0.001 : 0), geo[1] - (isMobile ? 0 : 0.002)]);
  setZoom(17);
};

<Map state={{ center, zoom }} onLoad={() => setMapLoaded(true)}>
  <PlaceMarks contacts contactsList={listContactsInfoParams} activeId={activeId} setActiveId={handleActiveIdChange} />
</Map>
```

### Плавающий контактный виджет

В правом нижнем углу — фиксированный контейнер с мгновенным доступом к телефону, VK, Telegram, OK, Dzen и email. Он раскрывается/сворачивается пружинной анимацией, слушает `pointerdown` и `scroll`, чтобы автоматически закрываться, и реагирует на `whileTap`, создавая тактильный отклик.

```tsx
const FloatingButton = () => {
  const [open, setOpen] = useState(false);
  const iconsRef = useRef<HTMLDivElement>(null);
  const triggerRef = useRef<HTMLImageElement>(null);
  const [targetHeight, setTargetHeight] = useState(0);

  useEffect(() => {
    const handlePointerDown = (event: Event) => {
      const target = event.target as Node | null;
      if (iconsRef.current && target && !iconsRef.current.contains(target)) {
        setOpen(false);
      }
    };
    document.addEventListener('pointerdown', handlePointerDown);
    document.addEventListener('scroll', handlePointerDown);
    return () => {
      document.removeEventListener('pointerdown', handlePointerDown);
      document.removeEventListener('scroll', handlePointerDown);
    };
  }, []);

  useEffect(() => {
    const el = iconsRef.current;
    if (!el) return;
    const raf = requestAnimationFrame(() => setTargetHeight(el.clientHeight || 0));
    return () => cancelAnimationFrame(raf);
  }, [open]);

  return (
    <motion.div
      initial={{ opacity: 0, scale: 1 }}
      animate={{
        opacity: 1,
        height: open ? targetHeight : triggerRef.current?.clientHeight,
        scaleY: open ? [2, 1] : 1,
        scaleX: open ? [0.5, 1] : 1,
      }}
      transition={{ type: 'spring', mass: 0.5 }}
      className="fixed bottom-5 right-5"
    >
      <div
        ref={iconsRef}
        onClick={() => { if (!open) setOpen(true); }}
        onMouseEnter={() => { if (!open) setOpen(true); }}
        className="grid grid-cols-1 gap-4"
      >
        {/* телефон + соцсети + кнопка закрытия */}
      </div>
    </motion.div>
  );
};
```

---

## Результат

- **Единый поток конверсии:** фуллскрин-слайдер → преимущества → интерактивные карты → каталоги → формы.
- **Премиальное ощущение без перегруза CPU:** Framer Motion, Swiper, Tailwind и блюр-превью изображений.
- **Ручные SVG-карты и схемы:** демонстрируют масштаб и технологичность продукции.
- **Готовность к росту:** архитектура (Next.js + TypeScript + Zustand + Tailwind) позволяет быстро добавлять направления и кампании без поломки UX.

