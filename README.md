# -
无
import os
import re
import time
import requests
from bs4 import BeautifulSoup
from concurrent.futures import ThreadPoolExecutor, as_completed
from urllib.parse import urljoin
import sys

class NovelScraper:
    def __init__(self):
        self.headers = {
            'User-Agent': 'Mozilla/5.0 (Linux; Android 10; SM-A505F) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.164 Mobile Safari/537.36',
            'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
            'Accept-Language': 'zh-CN,zh;q=0.9,en;q=0.8',
            'Connection': 'keep-alive',
            'Referer': 'https://www.2dsk.com/'
        }
        self.session = requests.Session()
        self.session.headers.update(self.headers)
        
    def get_novel_info(self, url):
        """获取小说基本信息"""
        print(f"正在获取小说信息: {url}")
        try:
            response = self.session.get(url, timeout=10)
            response.encoding = 'gbk'  # 网站使用GBK编码
            response.raise_for_status()
            soup = BeautifulSoup(response.text, 'html.parser')
            
            # 获取小说标题
            title_tag = soup.select_one('div.book-info > h1')
            title = title_tag.text.strip() if title_tag else "未知小说"
            
            # 获取作者
            author_tag = soup.select_one('div.book-info > div.info > span:nth-child(1) > a')
            author = author_tag.text.strip() if author_tag else "未知作者"
            
            # 获取封面图片
            cover_tag = soup.select_one('div.book-img > img')
            cover_url = cover_tag['src'] if cover_tag and 'src' in cover_tag.attrs else None
            
            # 获取小说简介
            intro_tag = soup.select_one('div.book-intro > p')
            intro = intro_tag.text.strip() if intro_tag else "暂无简介"
            
            # 获取章节链接
            chapter_links = []
            # 获取所有卷
            volumes = soup.select('div.book-chapter-list > div.volume')
            for volume in volumes:
                # 获取卷标题
                volume_title = volume.select_one('div.volume-title').text.strip()
                # 获取该卷下的章节
                chapters = volume.select('ul.chapter-list > li > a')
                for chapter in chapters:
                    href = chapter.get('href')
                    if href:
                        full_url = urljoin(url, href)
                        chapter_title = f"{volume_title} {chapter.text.strip()}"
                        chapter_links.append((full_url, chapter_title))
            
            print(f"发现 {len(chapter_links)} 个章节")
            return {
                'title': title,
                'author': author,
                'cover_url': cover_url,
                'intro': intro,
                'chapter_links': chapter_links,
                'base_url': url
            }
        except Exception as e:
            print(f"获取小说信息失败: {str(e)}")
            return None
    
    def get_chapter_content(self, url, chapter_title):
        """获取章节内容"""
        try:
            response = self.session.get(url, timeout=15)
            response.encoding = 'gbk'  # 网站使用GBK编码
            response.raise_for_status()
            soup = BeautifulSoup(response.text, 'html.parser')
            
            # 提取正文内容
            content_div = soup.find('div', id='content')
            if content_div:
                # 清理不需要的标签
                for script in content_div.find_all('script'):
                    script.decompose()
                for div in content_div.find_all('div', class_='ads'):
                    div.decompose()
                
                # 处理特殊标签
                for br in content_div.find_all('br'):
                    br.replace_with('\n')
                
                # 获取文本内容
                content = content_div.get_text()
                
                # 清理文本
                content = re.sub(r'\s*\n\s*', '\n', content)  # 清理多余空白
                content = re.sub(r'[\r\t]', '', content)  # 移除特殊字符
                content = re.sub(r'\n{3,}', '\n\n', content)  # 减少连续空行
                content = content.strip()
                
                # 添加章节标题
                content = f"{chapter_title}\n\n{content}"
            else:
                content = f"{chapter_title}\n\n未能提取本章内容"
            
            return {
                'title': chapter_title,
                'content': content,
                'url': url
            }
        except Exception as e:
            print(f"获取章节内容失败 ({url}): {str(e)}")
            return {
                'title': chapter_title,
                'content': f"{chapter_title}\n\n章节获取失败",
                'url': url
            }
    
    def download_novel(self, url, max_workers=3):
        """下载整本小说"""
        novel_info = self.get_novel_info(url)
        if not novel_info:
            print("无法获取小说信息，请检查URL是否正确")
            return False
        
        print(f"开始下载: 《{novel_info['title']}》 - 作者: {novel_info['author']}")
        print(f"小说简介: {novel_info['intro'][:100]}...")
        print(f"共发现 {len(novel_info['chapter_links'])} 章")
        
        # 创建保存目录
        output_dir = "novels"
        os.makedirs(output_dir, exist_ok=True)
        filename = os.path.join(output_dir, f"{novel_info['title']}.txt")
        
        # 多线程下载章节
        chapters = []
        completed = 0
        total_chapters = len(novel_info['chapter_links'])
        start_time = time.time()
        
        # 限制最大线程数（手机环境不宜过多）
        max_workers = min(max_workers, 5)
        
        with ThreadPoolExecutor(max_workers=max_workers) as executor:
            futures = []
            for chapter_url, chapter_title in novel_info['chapter_links']:
                future = executor.submit(self.get_chapter_content, chapter_url, chapter_title)
                futures.append(future)
            
            for future in as_completed(futures):
                try:
                    result = future.result()
                    if result:
                        chapters.append(result)
                        completed += 1
                        elapsed = time.time() - start_time
                        remaining = (elapsed / completed) * (total_chapters - completed) if completed > 0 else 0
                        print(f"\r进度: {completed}/{total_chapters} 章 | 耗时: {elapsed:.1f}s | 剩余: {remaining:.1f}s", end='', flush=True)
                except Exception as e:
                    print(f"\n章节下载失败: {str(e)}")
        
        # 按原始顺序排序章节
        chapters.sort(key=lambda x: novel_info['chapter_links'].index((x['url'], x['title'])))
        
        # 保存到文件
        with open(filename, 'w', encoding='utf-8') as f:
            # 写入元数据
            f.write(f"书名: 《{novel_info['title']}》\n")
            f.write(f"作者: {novel_info['author']}\n")
            f.write(f"源URL: {novel_info['base_url']}\n")
            if novel_info['cover_url']:
                f.write(f"封面: {novel_info['cover_url']}\n")
            f.write(f"下载时间: {time.strftime('%Y-%m-%d %H:%M:%S')}\n\n")
            
            # 写入简介
            f.write("【简介】\n")
            f.write(novel_info['intro'])
            f.write("\n\n")
            
            # 写入章节内容
            f.write("=" * 60 + "\n")
            f.write("正文开始\n")
            f.write("=" * 60 + "\n\n")
            
            for i, chapter in enumerate(chapters):
                f.write(chapter['content'])
                f.write("\n\n")
                
                # 每10章添加分隔线
                if (i + 1) % 10 == 0:
                    f.write("-" * 60 + "\n\n")
        
        print(f"\n\n下载完成！小说已保存到: {filename}")
        print(f"总章节数: {len(chapters)}")
        print(f"总耗时: {time.time() - start_time:.1f}秒")
        return True

def main():
    print("=" * 60)
    print("手机版2dsk小说网小说下载工具")
    print("=" * 60)
    
    # 示例URL
    example_url = "https://www.2dsk.com/book/149161.html"
    
    url = input(f"请输入小说目录页URL (直接回车使用示例): ").strip()
    if not url:
        url = example_url
        print(f"使用示例小说: {url}")
    
    # 线程数设置（手机环境限制在3线程）
    threads = 3
    
    scraper = NovelScraper()
    scraper.download_novel(url, max_workers=threads)

if __name__ == "__main__":
    main()
