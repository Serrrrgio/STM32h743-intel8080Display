# STM32h743-intel8080Display

Всех приветствую. Эта страничка создана для тех кто хотел бы подключить экраны по параллельному протоколу intel-8080 II. Для примера будем использовать экран на контроллере ili9341 и отладочную плату от stm32 nucleo STM32H743ZI2. Таких гайдов полно в интернете, однако,  опыт подсказывает, что для серии h7 нету, а если и есть, то обычно используют уже интерфейс LTDC. 

Для реализации этого протокола можно пойти 2 путями или самому дергать ножками (ногодрыг) или использовать FMC. В данной статье пойдет речь о втором варианте (первый нам не интересен). Для демонстрации будем использовать CubeMx. 
Выставляем все как на картинке (A16 – можно ставить, какой вам надо только не надо пометь адрес FMC). Адрес FMC будет для команд COM=0x60’00’00’00 для данных команд DATA= COM + 1<<(16+1). 

![1](https://user-images.githubusercontent.com/32983504/150678080-fc246594-1a09-45df-894e-6750e80609e0.png)

 Подключив все, как нужное и запустив драйвер экрана мы получим – НИЧЕГО. Подключаем управляющие контакты к логическому анализатору и видим, что у нас не выполняется тайминги, что написаны в даташите. И тут сразу возникает вопрос о их настройке. И тут мы подходим к вопросу, что это просто ни как не сделать. Есть вариант уменьшать частоту тактирования FMC, но по итогу это или не заработает или банально частота кадров будет очень печальной. 
 
 ![2](https://user-images.githubusercontent.com/32983504/150678121-d9b02d5a-e6ac-4c0a-8e70-a9ad9420aa99.png)

В ходе работы встречал ответы, что необходимо сделать ремап памяти в друю область однако, это мне не помогло. Поэтому опустим этот вариант.

Что же делать задаётесть вы вопросом! Попробуем пойдти другим путем. Настроим FMC для работы с NAND памятью. Там есть возможность выставлять тайминги более тонко.

![3](https://user-images.githubusercontent.com/32983504/150678566-f7012a98-f5a6-4f66-82da-415dd3d916cb.png)

Тактирование ставим, как мы все любим на максимум 480МГц!!!. FMC 200 МГц. Теперь адрес куда будем слыть команды или данные у нас другой это соотвественно 0x80'00'00'00 и 0x80'01'00'00. 

NOE - LCD !RD
NWE - LCD !WE
NCE - LCD !CS
A16 - CLE - LCD !RS
A17 - ALE - not used

Теперь настраиваем MPU

MPU_Region_InitTypeDef MPU_InitStruct;
 
HAL_MPU_Disable();
 
MPU_InitStruct.Enable=MPU_REGION_ENABLE;
 
MPU_InitStruct.BaseAddress = 0x80000000;

MPU_InitStruct.Size = MPU_REGION_SIZE_256MB; //0x80000000..0x8FFFFFFF

MPU_InitStruct.AccessPermission=MPU_REGION_FULL_ACCESS;
 
MPU_InitStruct.TypeExtField=MPU_TEX_LEVEL0;
 
MPU_InitStruct.IsCacheable=MPU_ACCESS_NOT_CACHEABLE; //NOT CACHEABLE

MPU_InitStruct.IsBufferable=MPU_ACCESS_BUFFERABLE;

MPU_InitStruct.IsShareable=MPU_ACCESS_SHAREABLE;
 
MPU_InitStruct.Number=MPU_REGION_NUMBER0;

MPU_InitStruct.SubRegionDisable=0x00;

MPU_InitStruct.DisableExec=MPU_INSTRUCTION_ACCESS_DISABLE;
 
HAL_MPU_ConfigRegion(&MPU_InitStruct);
 
HAL_MPU_Enable(MPU_PRIVILEGED_DEFAULT);

И тайминги FMC как у меня на картинке

  FMC_NAND_PCC_TimingTypeDef ComSpaceTiming;
  
  FMC_NAND_PCC_TimingTypeDef AttSpaceTiming;
 
  /** Perform the NAND1 memory initialization sequence
  */
  
  hnand1.Instance = FMC_NAND_DEVICE;
  
  /* hnand1.Init */
  hnand1.Init.NandBank = FMC_NAND_BANK3;
  
  hnand1.Init.Waitfeature = FMC_NAND_WAIT_FEATURE_DISABLE;
  
  hnand1.Init.MemoryDataWidth = FMC_NAND_MEM_BUS_WIDTH_8;
  
  hnand1.Init.EccComputation = FMC_NAND_ECC_DISABLE;
  
  hnand1.Init.ECCPageSize = FMC_NAND_ECC_PAGE_SIZE_256BYTE;
  
  hnand1.Init.TCLRSetupTime = 0;
  
  hnand1.Init.TARSetupTime = 0;
  
  /* hnand1.Config */
  
  hnand1.Config.PageSize      = 262144; //256kB >= 320x240x2
  
  hnand1.Config.SpareAreaSize = 262144; //256kB >= 320x240x2
  
  hnand1.Config.BlockSize = 1;
  
  hnand1.Config.BlockNbr = 1;
  
  hnand1.Config.PlaneNbr = 1;
  
  hnand1.Config.PlaneSize = 1;
  
  hnand1.Config.ExtraCommandEnable = ENABLE;
  
 
//Адреса для записи:

  /* ComSpaceTiming: 0x80000000..0x83FFFFFF */
  
  ComSpaceTiming.SetupTime     = 1; //+1 1CLK =  5ns             >=  4ns
  
  ComSpaceTiming.WaitSetupTime = 4; //+1 4CLK = 20ns             >= 18ns 1CLK уже мало
  
  ComSpaceTiming.HoldSetupTime = 5; //+0 3CLK = 15ns             >= 15ns 1CLK уже мало
  
  ComSpaceTiming.HiZSetupTime  = 0; //+0 1CLK =  5ns (tSET=tHIZ) >=  4ns 0CLK работает (0CLK: данные появятся сразу после падения NCE, 1CLK: после падения NWE, >1CLK: ещё позже)
 
//Адреса для чтения:
  /* AttSpaceTiming: 0x88000000..0x8BFFFFFF */
  AttSpaceTiming.SetupTime     = 254; //+1
  
  AttSpaceTiming.WaitSetupTime = 254; //+1
  
  AttSpaceTiming.HoldSetupTime = 254; //WR:+0 RD:+1
  
  AttSpaceTiming.HiZSetupTime  = 254; //+0
 
  if (HAL_NAND_Init(&hnand1, &ComSpaceTiming, &AttSpaceTiming) != HAL_OK)
  {
    _Error_Handler(__FILE__, __LINE__);
  }

Прошиваем и радуемся результату
![1642422428162](https://user-images.githubusercontent.com/32983504/150679111-20f2fca1-f2f6-4969-8555-5a818179e146.jpg)

