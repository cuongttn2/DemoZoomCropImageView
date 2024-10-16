// bug: draw

@SuppressLint("ClickableViewAccessibility")
class PopkitZoomCropView @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = 0,
) : ImageView(context, attrs, defStyleAttr) {

    private var _matrix = Matrix()
    private var _mode = NONE_MODE
    private var _previousPoint = PointF()
    private var _currentScale = 1f
    private var _minScale = 1f
    private var _maxScale = 3f
    private lateinit var _scaleDetector: ScaleGestureDetector

    private val path = Path()
    private lateinit var maskBitmap: Bitmap
    private lateinit var maskCanvas: Canvas
    private var originalBitmap: Bitmap? = null

    private val paint = Paint().apply {
        color = Color.RED // Đường vẽ màu đỏ
        style = Paint.Style.STROKE
        strokeWidth = 50f
        isAntiAlias = true
    }

    init {
        setLayerType(LAYER_TYPE_SOFTWARE, null) // Để hỗ trợ PorterDuff xfermode
        scaleType = ScaleType.MATRIX // Sử dụng Matrix để xử lý zoom và kéo
        _scaleDetector =
            ScaleGestureDetector(context, ScaleListener()) // Khởi tạo ScaleGestureDetector
        setOnTouchListener { _, event -> handleTouch(event) }
    }

    fun setImage(bitmap: Bitmap) {
        originalBitmap = bitmap

        // Sử dụng post() để đảm bảo view đã được layout xong
        post {
            val viewWidth = width
            if (viewWidth > 0) {
                // Tính toán chiều cao mới dựa trên tỷ lệ của bitmap
                val bitmapWidth = bitmap.width
                val bitmapHeight = bitmap.height
                val newHeight = (viewWidth.toFloat() / bitmapWidth.toFloat() * bitmapHeight).toInt()

                // Cập nhật chiều cao của PopkitZoomCropView
                val layoutParams = layoutParams
                layoutParams.height = newHeight
                this.layoutParams = layoutParams

                // Đặt lại bitmap đã được điều chỉnh tỷ lệ
                val scaledBitmap = Bitmap.createScaledBitmap(bitmap, viewWidth, newHeight, true)
                setImageBitmap(scaledBitmap)

                // Khởi tạo lại `maskBitmap` và `maskCanvas` với kích thước phù hợp với view
                maskBitmap = Bitmap.createBitmap(viewWidth, newHeight, Bitmap.Config.ARGB_8888)
                maskCanvas = Canvas(maskBitmap)
                maskCanvas.drawColor(Color.TRANSPARENT)

                // Yêu cầu vẽ lại view
                invalidate()
            }
        }
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)

        // Vẽ `maskBitmap` lên canvas của view để đảm bảo các đường vẽ được hiển thị
        if (::maskBitmap.isInitialized) {
            canvas.drawBitmap(maskBitmap, 0f, 0f, null)
        }
    }

    private fun handleTouch(event: MotionEvent): Boolean {
        _scaleDetector.onTouchEvent(event)

        when (event.action and MotionEvent.ACTION_MASK) {
            MotionEvent.ACTION_DOWN -> {
                _previousPoint.set(event.x, event.y)
                path.moveTo(event.x, event.y)
                _mode = DRAG_MODE
            }

            MotionEvent.ACTION_MOVE -> {
                if (_mode == ZOOM_MODE) {
                    // Xử lý kéo ảnh dựa trên tọa độ ngón tay
                    val deltaX = event.x - _previousPoint.x
                    val deltaY = event.y - _previousPoint.y
                    _matrix.postTranslate(deltaX, deltaY)
                    imageMatrix = _matrix
                    invalidate() // Cập nhật lại view sau khi kéo
                } else if (_mode == DRAG_MODE) {
                    // Chuyển đổi tọa độ để tương thích với tỷ lệ của `Matrix`
                    val invertedMatrix = Matrix()
                    _matrix.invert(invertedMatrix)

                    val mappedPoints = floatArrayOf(event.x, event.y)
                    invertedMatrix.mapPoints(mappedPoints)

                    path.lineTo(mappedPoints[0], mappedPoints[1])
                    maskCanvas.drawPath(path, paint) // Vẽ ngay lên `maskBitmap`

                    invalidate() // Vẽ lại view với đường mới
                }
                _previousPoint.set(event.x, event.y)
            }

            MotionEvent.ACTION_POINTER_DOWN -> {
                _mode = ZOOM_MODE
            }

            MotionEvent.ACTION_UP -> {
                if (_mode == DRAG_MODE) {
                    maskCanvas.drawPath(path, paint) // Vẽ `path` hiện tại lên `maskBitmap`
                    path.reset() // Reset `path` sau khi đã lưu vào `maskBitmap`
                }
                invalidate() // Vẽ lại view sau khi kết thúc thao tác
                _mode = NONE_MODE
            }
        }
        return true
    }

    // Hàm để crop vùng đã tô
    fun crop(): Bitmap {
        val original = originalBitmap ?: return maskBitmap

        // Tạo bitmap mới để lưu kết quả cắt
        val croppedBitmap =
            Bitmap.createBitmap(original.width, original.height, Bitmap.Config.ARGB_8888)
        val cropCanvas = Canvas(croppedBitmap)

        // Vẽ ảnh gốc lên canvas mới
        cropCanvas.drawBitmap(original, 0f, 0f, null)

        // Tạo Paint với chế độ PorterDuff.Mode.DST_IN để chỉ giữ lại phần đã tô
        val maskPaint = Paint().apply {
            xfermode = PorterDuffXfermode(PorterDuff.Mode.DST_IN)
            isAntiAlias = true
        }

        // Áp dụng mặt nạ lên ảnh gốc
        cropCanvas.drawBitmap(maskBitmap, 0f, 0f, maskPaint)

        return croppedBitmap
    }

    fun clearMask() {
        path.reset()
        maskBitmap.eraseColor(Color.TRANSPARENT) // Xóa mặt nạ
        invalidate() // Vẽ lại view
    }

    fun setMaxZoom(maxZoom: Float) {
        _maxScale = maxZoom
    }

    fun setMinZoom(minZoom: Float) {
        _minScale = minZoom
    }

    private inner class ScaleListener : ScaleGestureDetector.SimpleOnScaleGestureListener() {
        override fun onScale(detector: ScaleGestureDetector): Boolean {
            val scaleFactor = detector.scaleFactor
            val originalScale = _currentScale
            _currentScale *= scaleFactor

            // Giới hạn zoom
            if (_currentScale > _maxScale) {
                _currentScale = _maxScale
            } else if (_currentScale < _minScale) {
                _currentScale = _minScale
            }

            val scale = _currentScale / originalScale
            _matrix.postScale(scale, scale, detector.focusX, detector.focusY)
            imageMatrix = _matrix
            invalidate() // Cập nhật lại view để áp dụng thay đổi
            return true
        }
    }

    companion object {
        const val NONE_MODE = 0
        const val DRAG_MODE = 1
        const val ZOOM_MODE = 2
    }
}
