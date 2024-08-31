# BMP Loader

```lua
local BMP = {}

---@param file file*
local function readbyte(file)
	local byte = file:read(1)
	if byte then
		return string.byte(byte)
	end
	error('Failed to read a data from the file stream!')
end

---@param file file*
local function readuint16(file)
	local bytes = file:read(2)
	if bytes then
		local a, b = string.byte(bytes, 1, 2)
		return b * 256 + a
	end
	error('Failed to read a data from the file stream!')
end

---@param file file*
local function readint16(file)
	local bytes = file:read(2)
	if bytes then
		local a, b = string.byte(bytes, 1, 2)
		local value = b * 256 + a
		if value >= 32768 then
			value = value - 65536
		end
		return value
	end
	error('Failed to read a data from the file stream!')
end

---@param file file*
local function readuint32(file)
	local bytes = file:read(4)
	if bytes then
		local a, b, c, d = string.byte(bytes, 1, 4)
		return d * 16777216 + c * 65536 + b * 256 + a
	end
	error('Failed to read a data from the file stream!')
end

---@param file file*
local function readint32(file)
	local bytes = file:read(4)
	if bytes then
		local a, b, c, d = string.byte(bytes, 1, 4)
		local value = d * 16777216 + c * 65536 + b * 256 + a
		if value >= 2147483648 then
			value = value - 4294967296
		end
		return value
	end
	error('Failed to read a data from the file stream!')
end

---@class Reader
local reader = {
	---@type file*
	file = nil,
	---@param self Reader
	readByte = function(self) return readbyte(self.file) end,
	---@param self Reader
	readUInt16 = function(self) return readuint16(self.file) end,
	---@param self Reader
	readInt16 = function(self) return readint16(self.file) end,
	---@param self Reader
	readUInt32 = function(self) return readuint32(self.file) end,
	---@param self Reader
	readInt32 = function(self) return readint32(self.file) end,
	---@param self Reader
	---@param size integer
	readBytes = function(self, size) return self.file:read(size) end,
	---@param self Reader
	available = function(self) return self.file:seek("end") - self.file:seek() end,
	---@param self Reader
	---@param size integer
	skip = function(self, size) self.file:seek("cur", size) end,
}

---@param filename string path to the file to load
---@param image_open_method nil | fun(filename:string):file* a closure that returns the file object
function BMP.Load(filename, image_open_method)
	local image_open_method = image_open_method or function(filename) return io.open(filename, "rb") end
	local file = image_open_method(filename)
	if not file then
		error("Failed to open file: " .. filename)
	end
	reader.file = file

	---@class BmpImage
	local bmp = {
		_file = file,
		_reader = reader,
		_info = {},
		---@type table<integer, integer>
		_imageData = nil,
		---@param self BmpImage
		get_width = function(self)
			return self._info.width
		end,
		---@param self BmpImage
		get_height = function(self)
			return self._info.height
		end,
		---@param self BmpImage
		---@param x integer The x coordinate of the pixel. 1-based, like Lua
		---@param y integer The y coordinate of the pixel. 1-based, like Lua
		---@param r integer Color, 0-255 based
		---@param g integer Color, 0-255 based
		---@param b integer Color, 0-255 based
		---@param a integer Color, 0-255 based
		set_pixel = function(self, x, y, r, g, b, a)
			local w = self:get_width()
			local rx = x - 1
			local ry = y - 1
			local idx = 1 + (rx + ry * w) * 4
			local d = self._imageData
			d[idx + 0] = r
			d[idx + 1] = g
			d[idx + 2] = b
			d[idx + 3] = a
		end,
		---@param self BmpImage
		---@param x integer The x coordinate of the pixel. 1-based, like Lua
		---@param y integer The y coordinate of the pixel. 1-based, like Lua
		---@return number r Color, 0-255 based
		---@return number g Color, 0-255 based
		---@return number b Color, 0-255 based
		---@return number a Color, 0-255 based
		get_pixel = function(self, x, y)
			local w = self:get_width()
			local rx = x - 1
			local ry = y - 1
			local idx = 1 + (rx + ry * w) * 4
			local d = self._imageData
			return d[idx + 0],
				d[idx + 1],
				d[idx + 2],
				d[idx + 3]
		end
	}

	-- Read file header
	bmp._info.magic = reader:readUInt16()
	if bmp._info.magic ~= 0x4D42 then
		error("Not a BMP file")
	end

	bmp._info.filesize = reader:readUInt32()
	bmp._info.reserved = reader:readUInt32()
	bmp._info.offset = reader:readUInt32()

	-- Read info header
	bmp._info.size = reader:readUInt32()
	if bmp._info.size < 40 then
		error("Invalid BMP info header size")
	end

	bmp._info.width = reader:readInt32()
	bmp._info.height = reader:readInt32()
	bmp._info.nColorPlanes = reader:readUInt16()
	bmp._info.nBitsPerPixel = reader:readUInt16()
	bmp._info.compressionMethod = reader:readInt32()
	bmp._info.rawImageSize = reader:readUInt32()
	bmp._info.xPPM = reader:readInt32()
	bmp._info.yPPM = reader:readInt32()
	bmp._info.nPaletteColors = reader:readUInt32()
	bmp._info.nImportantColors = reader:readUInt32()

	local compressed = bmp._info.compressionMethod ~= 0
	-- Check palette
	if bmp._info.nBitsPerPixel ~= 32 and bmp._info.nBitsPerPixel ~= 24 then
		error("Images with palette information aren't supported yet!")
	end

	-- Read image data
	if compressed then
		error("Unsupported BMP compression method")
	else
		bmp._imageData = BMP.Load32BitImage(reader, bmp)
	end

	bmp._file:close()
	bmp._file = nil
	reader.file = nil
	return bmp
end

function BMP.Load32BitImage(reader, bmp)
	local w, h = bmp._info.width, bmp._info.height
	local data = {}

	local pixel_width = 3
	if bmp._info.nBitsPerPixel == 32 then
		pixel_width = 4
	end

	local counter = 0
	for y = 1, h do
		for x = 1, w do
			counter = counter + 4
		end
	end

	for y = 1, h do
		local ry = y - 1
		for x = 1, w do
			local r, g, b, a = reader:readByte(), reader:readByte(), reader:readByte(), 255
			if pixel_width == 4 then
				a = reader:readByte()
			end
			local rx = x - 1

			assert(r ~= nil and g ~= nil and b ~= nil and a ~= nil, "Values returned by the bmp loader shouldn't be nil")

			assert(
				data[1 + (rx + ry * w) * 4 + 0] == nil and
				data[1 + (rx + ry * w) * 4 + 1] == nil and
				data[1 + (rx + ry * w) * 4 + 2] == nil and
				data[1 + (rx + ry * w) * 4 + 3] == nil,
				"There shouldn't be any non nil entries in the data table as it's being loaded."
			)

			data[1 + (rx + ry * w) * 4 + 0] = r
			data[1 + (rx + ry * w) * 4 + 1] = g
			data[1 + (rx + ry * w) * 4 + 2] = b
			data[1 + (rx + ry * w) * 4 + 3] = a

			assert(
				data[1 + (rx + ry * w) * 4 + 0] ~= nil and
				data[1 + (rx + ry * w) * 4 + 1] ~= nil and
				data[1 + (rx + ry * w) * 4 + 2] ~= nil and
				data[1 + (rx + ry * w) * 4 + 3] ~= nil,
				"After writing colors, nulls should be gone."
			)
		end
	end

	return data
end

return BMP
```
