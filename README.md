# AD983X Driver usage

```c
/**
 * @struct AD983X
 * @brief Represents the configuration and state of an AD983X waveform generator device.
 *
 * This structure is used to manage the SPI communication, GPIO control, and internal
 * state of an AD983X device, such as the AD9833 or AD9834, which are programmable
 * waveform generators.
 *
 * @var AD983X::hspi
 * Pointer to the SPI handle used for communication with the AD983X device.
 *
 * @var AD983X::m_select_port
 * GPIO port used for the chip select (CS) signal to communicate with the AD983X device.
 *
 * @var AD983X::m_select_pin
 * GPIO pin used for the chip select (CS) signal to communicate with the AD983X device.
 *
 * @var AD983X::m_reset_port
 * GPIO port used for the reset signal to control the AD983X device.
 *
 * @var AD983X::m_reset_pin
 * GPIO pin used for the reset signal to control the AD983X device.
 *
 * @var AD983X::m_reg
 * Internal register value used to store the current configuration of the AD983X device.
 *
 * @var AD983X::m_clk_scaler
 * Clock scaler value used to calculate the output frequency of the AD983X device.
 */
struct AD983X {
  SPI_HandleTypeDef *hspi;
  GPIO_TypeDef *m_select_port;
  uint16_t m_select_pin;
  GPIO_TypeDef *m_reset_port;
  uint16_t m_reset_pin;
  uint16_t m_reg;
  double m_clk_scaler;
}

/**
 * @brief Initializes the AD983X waveform generator device.
 *
 * This function sets up the AD983X device by configuring the SPI interface,
 * GPIO pins for chip select and reset, and calculating the clock scaler
 * based on the provided clock frequency. It also performs an initial reset
 * and writes a default configuration to the device.
 *
 * @param self Pointer to the AD983X instance structure.
 * @param hspi Pointer to the SPI handle used for communication with the device.
 * @param select_port GPIO port for the chip select (CS) pin.
 * @param select_pin GPIO pin number for the chip select (CS) pin.
 * @param reset_port GPIO port for the reset pin.
 * @param reset_pin GPIO pin number for the reset pin.
 * @param clk_mhz Clock frequency of the AD983X device in MHz.
 */
void AD983X_init(AD983X *self, SPI_HandleTypeDef *hspi,
                 GPIO_TypeDef *select_port, uint16_t select_pin,
                 GPIO_TypeDef *reset_port, uint16_t reset_pin, uint8_t clk_mhz);

/**
 * @brief Sets the output waveform of the AD983X device.
 *
 * This function configures the AD983X device to output a specific waveform
 * based on the provided mode. The available modes correspond to different
 * waveform types such as sine, triangle, and square waves.
 *
 * @param self Pointer to the AD983X device instance.
 * @param mode The waveform mode to set:
 *             - 0: Sine wave
 *             - 1: Triangle wave
 *             - 2: Square wave
 *
 * @note Ensure that the mode parameter is within the valid range (0-2).
 *       Passing an invalid mode may result in undefined behavior.
 */
void AD983X_setOutputWave(AD983X *self, uint8_t mode);

/**
 * @brief Sets the output frequency of the AD983X device.
 *
 * This function configures the AD983X device to output a specific frequency
 * by writing the appropriate frequency word to the specified frequency
 * register. It temporarily sets the device to continuous write mode, updates
 * the frequency, and then restores the device to its normal operating mode.
 *
 * @param self Pointer to the AD983X device instance.
 * @param reg The frequency register to update (e.g., 0 for FREQ0, 1 for FREQ1).
 * @param frequency The desired output frequency in Hz.
 */
void AD983X_setFrequency(AD983X *self, uint8_t reg, double frequency);
```