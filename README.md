# videoplayer
基于mpi的并行视频播放器
#include <stdio.h>
#include <libavformat/avformat.h>
#include <libavcodec/avcodec.h>
#include <libswscale/swscale.h>
#include <libavutil/imgutils.h>


#include "SDL2/SDL.h"
#include <mpi.h>

#include<libavutil/eval.h>

#define bpp 24//位图
#define wh2 1138
int screen_w=1280,screen_h=720;
#define pixel_w 1280
#define pixel_h 720

char buf[pixel_w*pixel_h*bpp/8];


//Refresh Event
#define REFRESH_EVENT  (SDL_USEREVENT + 1)

#define BREAK_EVENT  (SDL_USEREVENT + 2)

int thread_exit=0;

int refresh_video(void *opaque){
	thread_exit=0;
	while (!thread_exit) {
		SDL_Event event;
		event.type = REFRESH_EVENT;
		SDL_PushEvent(&event);
		SDL_Delay(15);
	}
	thread_exit=0;
	//Break
	SDL_Event event;
	event.type = BREAK_EVENT;
	SDL_PushEvent(&event);

	return 0;
}

int cmd(char* cmd, char* result)
{
    char buffer[10240];
    FILE* pipe = popen(cmd, "r");
    if (!pipe)
    return -1;
    while(!feof(pipe)) {
        if(fgets(buffer, 4096, pipe)){
            strcat(result, buffer);
        }
    }
    pclose(pipe);
    return 0;
}

void bfq(){
      int zs=3;
      char *Filename[100];
	if(SDL_Init(SDL_INIT_VIDEO)) {
		printf( "Could not initialize SDL - %s\n", SDL_GetError());
		return -1;
	}

	SDL_Window *screen;
	//SDL 2.0 Support for multiple windows
	screen = SDL_CreateWindow("Simplest Video Play SDL2", SDL_WINDOWPOS_UNDEFINED, SDL_WINDOWPOS_UNDEFINED,
		screen_w, screen_h,SDL_WINDOW_OPENGL|SDL_WINDOW_RESIZABLE);
	if(!screen) {
		printf("SDL: could not create window - exiting:%s\n",SDL_GetError());
		return -1;
	}
	SDL_Renderer* sdlRenderer = SDL_CreateRenderer(screen, -1, 0);

	Uint32 pixformat=0;

	//IYUV: Y + U + V  (3 planes)
	//YV12: Y + V + U  (3 planes)
	pixformat= SDL_PIXELFORMAT_IYUV;

	SDL_Texture* sdlTexture = SDL_CreateTexture(sdlRenderer,pixformat, SDL_TEXTUREACCESS_STREAMING,pixel_w,pixel_h);

	FILE *fp=NULL;
	SDL_Rect sdlRect;

	SDL_Thread *refresh_thread = SDL_CreateThread(refresh_video,NULL,NULL);
	SDL_Event event;
	while(1){
		//Wait
		sprintf(Filename,"/home/mpi_install/mpich-3.3.2/m/frame-%d.yuv",zs);
		fp=fopen(Filename,"rb+");
			if(fp==NULL){
		break;
	}
		SDL_WaitEvent(&event);
		if(event.type==REFRESH_EVENT){
			if (fread(buf, 1, pixel_w*pixel_h*bpp/8, fp) != pixel_w*pixel_h*bpp/8){
				// Loop
				fseek(fp, 0, SEEK_SET);
				fread(buf, 1, pixel_w*pixel_h*bpp/8, fp);
			}

			SDL_UpdateTexture( sdlTexture, NULL, buf, pixel_w);

			//FIX: If window is resize
			sdlRect.x = 0;
			sdlRect.y = 0;
			sdlRect.w = screen_w;
			sdlRect.h = screen_h;

			SDL_RenderClear( sdlRenderer );
			SDL_RenderCopy( sdlRenderer, sdlTexture, NULL, &sdlRect);
			SDL_RenderPresent( sdlRenderer );

		}else if(event.type==SDL_WINDOWEVENT){
			//If Resize
			SDL_GetWindowSize(screen,&screen_w,&screen_h);
		}else if(event.type==SDL_QUIT){
			thread_exit=1;
		}else if(event.type==BREAK_EVENT){
			break;
		}
		zs++;
	}
	SDL_Quit();
}

int main(int argc, char *argv[])
{
     int tag=0;
int bj[3];
    char buffer[10240]="";

    int rank, size, i,message[3];
    MPI_Status status;

    MPI_Init(&argc, &argv);//启动

    MPI_Comm_rank(MPI_COMM_WORLD, &rank);//查找进程号

    MPI_Comm_size(MPI_COMM_WORLD, &size);//查找进程总数
    if (rank==0)
    {
        if(cmd("ffprobe -v error -count_frames -select_streams v:0 -show_entries stream=nb_read_frames -of default=nokey=1:noprint_wrappers=1 1.h264", buffer) == 0)
        printf("cmd output : %s\n", buffer);
    int zzs=atoi(buffer);
          int wh_jc=size-1;
		   printf("jc %d",wh_jc);
          int add=zzs/wh_jc;//each jc需要处理的帧
          int i=1;
          printf("进程总数：%d\n",size);
          while(i<(size))
          {
          message[1]=add;
  	  message[2]=i;//序列号
          MPI_Send(message,2,MPI_INT,i,tag,MPI_COMM_WORLD);
          i++;
          }
          int wh3=0;
          int fc=(zzs%(size-1))+1;
          while(wh3==0){
          MPI_Recv(bj,1,MPI_INT,fc,7,MPI_COMM_WORLD,&status);
           
          if(bj[0]==wh2){
          bfq();
          return 0; 
          }
          
          }
    }
    else {
     char msg[4];
    int R=rank;
    MPI_Recv(message,2,MPI_INT,0,tag,MPI_COMM_WORLD,&status);
    printf("myrank:%d\n",R);
    int xlh=message[2];
    printf("序列号%d\n",R);
    int p=message[1];
     printf("%d\n",p);

        AVFormatContext	*pFormatCtx;
	int				i, videoindex;
	AVCodecContext	*pCodecCtx;
	AVCodec			*pCodec;
	AVFrame	*pFrame,*pFrameYUV;
	unsigned char *out_buffer;
	AVPacket *packet;
	int y_size;
	int ret, got_picture;
	struct SwsContext *img_convert_ctx;

	char filepath[]="1.h264";
        char Filename[32];

	//FILE *fp_yuv=fopen("1.yuv","wb+");

	av_register_all();
	avformat_network_init();
	pFormatCtx = avformat_alloc_context();

	if(avformat_open_input(&pFormatCtx,filepath,NULL,NULL)!=0){
		printf("Couldn't open input stream.\n");
		return -1;
	}
	if(avformat_find_stream_info(pFormatCtx,NULL)<0){
		printf("Couldn't find stream information.\n");
		return -1;
	}
	videoindex=-1;
	for(i=0; i<pFormatCtx->nb_streams; i++)
		if(pFormatCtx->streams[i]->codec->codec_type==AVMEDIA_TYPE_VIDEO){
			videoindex=i;
			break;
		}

	if(videoindex==-1){
		printf("Didn't find a video stream.\n");
		return -1;
	}

	pCodecCtx=pFormatCtx->streams[videoindex]->codec;
	pCodec=avcodec_find_decoder(pCodecCtx->codec_id);
	if(pCodec==NULL){
		printf("Codec not found.\n");
		return -1;
	}
	if(avcodec_open2(pCodecCtx, pCodec,NULL)<0){
		printf("Could not open codec.\n");
		return -1;
	}


	pFrame=av_frame_alloc();
	pFrameYUV=av_frame_alloc();
	out_buffer=(unsigned char *)av_malloc(av_image_get_buffer_size(AV_PIX_FMT_YUV420P,  pCodecCtx->width, pCodecCtx->height,1));
	av_image_fill_arrays(pFrameYUV->data, pFrameYUV->linesize,out_buffer,
		AV_PIX_FMT_YUV420P,pCodecCtx->width, pCodecCtx->height,1);



	packet=(AVPacket *)av_malloc(sizeof(AVPacket));
	//Output Info-----------------------------
	printf("--------------- File Information ----------------\n");
	av_dump_format(pFormatCtx,0,filepath,0);
	printf("-------------------------------------------------\n");
	img_convert_ctx = sws_getContext(pCodecCtx->width, pCodecCtx->height, pCodecCtx->pix_fmt,
		pCodecCtx->width, pCodecCtx->height, AV_PIX_FMT_YUV420P, SWS_BICUBIC, NULL, NULL, NULL);
        int zs=0;
        int wh=0;
	while(av_read_frame(pFormatCtx, packet)>=0){
	    wh++;
	    if(packet->stream_index==videoindex){
	    ret = avcodec_decode_video2(pCodecCtx, pFrame, &got_picture, packet);
	    if(ret < 0){
				printf("Decode Error.\n");
				return -1;
			}
			if(got_picture){
				sws_scale(img_convert_ctx, (const unsigned char* const*)pFrame->data, pFrame->linesize, 0, pCodecCtx->height,
					pFrameYUV->data, pFrameYUV->linesize);
					      if(wh%(size-1)==(R-1)){
                                sprintf(Filename,"/home/mpi_install/mpich-3.3.2/m/frame-%d.yuv",wh);
                                FILE *fp_yuv=fopen(Filename,"wb+");
				                y_size=pCodecCtx->width*pCodecCtx->height;
				                fwrite(pFrameYUV->data[0],1,y_size,fp_yuv);    //Y
                                fwrite(pFrameYUV->data[1],1,y_size/4,fp_yuv);  //U
				                fwrite(pFrameYUV->data[2],1,y_size/4,fp_yuv);  //v
								  fclose(fp_yuv);
								  printf("进程%dSucceed to decode %d frame!\n",R,wh);
                                        
	   if(wh==wh2){
             bj[0]=wh;
	      MPI_Send(bj,1,MPI_INT,0,7,MPI_COMM_WORLD);
               
          }
						  }



			}
			av_free_packet(packet);
		}
	}
	sws_freeContext(img_convert_ctx);


	av_frame_free(&pFrameYUV);
	av_frame_free(&pFrame);
	avcodec_close(pCodecCtx);
	avformat_close_input(&pFormatCtx);


    }
    printf("succeed----------------------------------!\n");
    MPI_Finalize();


          printf("succeed!\n");




}
