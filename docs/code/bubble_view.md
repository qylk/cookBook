微信聊天气泡-图片
```java
public class XORView extends View {
    Paint paint;
    Path path;
    BitmapShader brush;
    public XORView(Context context, AttributeSet attrs) {
        super(context, attrs);
        paint = new Paint();
        path = new Path();
        BitmapDrawable drawable = (BitmapDrawable)getResources().getDrawable(R.drawable.ic_texture);
        brush = new BitmapShader(drawable.getBitmap(), Shader.TileMode.REPEAT, Shader.TileMode.REPEAT);
    }
 
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        canvas.save();
        paint.setStyle(Paint.Style.FILL);
        //paint.setColor(Color.GREEN);
        paint.setShader(brush);
        canvas.drawPath(path, paint);
        paint.reset();
        paint.setStyle(Paint.Style.STROKE);
        canvas.drawPath(path, paint);
        canvas.restore();
    }
 
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        path.reset();
        int w = getWidth();
        int h = getHeight();
        path.moveTo(0,10);
        path.arcTo(new RectF(0,0,20,20),180,90);
        path.lineTo(w - 30,0);
        path.arcTo(new RectF(w - 40, 0, w - 20, 20), -90, 90);
        path.lineTo(w-20,h/2 -10);
        path.lineTo(w,h/2);
        path.lineTo(w-20,h/2 +10);
        path.lineTo(w-20,h - 10);
        path.arcTo(new RectF(w-40, h-20,w-20,h-1),0,90);
        path.lineTo(10,h-1);
        path.arcTo(new RectF(0,h-20,20,h-1),90,90);
        path.close();
    }
}
```

效果图：

![bubble](../assets/14.png)