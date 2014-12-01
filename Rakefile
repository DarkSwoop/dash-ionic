require 'bundler/setup'
require 'shellwords'
require 'json'
require 'uri'
Bundler.require :default

docset_root = "dist/Ionic.docset"
resources_path = "#{docset_root}/Contents/Resources"

class Generator
  class Link
    MAPPING = {
      "service" => 'Service',
      "directive" => 'Directive',
      "controller" => 'Object',
      "provider" => 'Provider',
      "utility" => 'Global',
      "object" => 'Constant',
      "page" => 'Component'
    }

    attr_accessor :href, :text

    def initialize(href, text, type=nil)
      @href = href
      @text = text
      @type = type unless type.nil?
    end

    def type
      return @type if @type
      type = href.match(/^.*?\/api\/(.*?)\/.*/)
      @type = MAPPING[type[1]] if type
    end

  end

  def initialize(database_path, file_path, section_selector, link_selector, default_type=nil)
    @file_path        = file_path
    @page             = Nokogiri::HTML(open(file_path))
    @section_selector = section_selector
    @link_selector    = link_selector
    @default_type     = default_type
    @db               = SQLite3::Database.new database_path
    setup_database
  end

  def setup_database
    rows = @db.execute <<-SQL
      CREATE TABLE IF NOT EXISTS searchIndex(id INTEGER PRIMARY KEY, name TEXT, type TEXT, path TEXT);
      CREATE UNIQUE INDEX IF NOT EXISTS anchor ON searchIndex (name, type, path);
    SQL
  end

  def parse
    categories = collect_links(@section_selector, "Guide")
    links      = collect_links(@link_selector, @default_type)

    link_collection = categories + links
    link_collection.each { |link| create_entry(link.text, link.type, link.href) }
  end

  def create_css_entry
    create_entry("Stylesheets", "Section", "docs/components/index.html")
  end

  def build_css_tree
    page = Nokogiri::HTML(open("compiled/docs/components/index.html"))
    sections = page.css("section.docs-section")
    sections.each do |node|

      main_section = node.css("> h2").first
      unless main_section.nil?
        add_docset_link(page, main_section, main_section.text)
      end

      section = node.css("h3.title").first
      unless section.nil?
        add_docset_link(page, section, section.text)
      end
    end

    File.open("compiled/docs/components/index.html", "w") do |f|
      f.write(page.to_html)
    end
  end

  private

  def add_docset_link(page, node, title)
    docset_link = Nokogiri::XML::Node.new "a", page
    docset_link["class"] = "dashAnchor"
    docset_link["name"] = "//apple_ref/cpp/Section/#{URI.escape(title)}"
    node.add_next_sibling(docset_link)
  end

  def create_entry(name, type, path)
    return if /\/?#/.match(path)
    path.gsub!("/#", "/index.html#")
    path.gsub!(/^\//, '')
    @db.execute "INSERT OR IGNORE INTO searchIndex(name, type, path) VALUES ( ?, ?, ?)", [name, type, path]
  end

  def collect_links(selector, type=nil)
    links = @page.css(selector)

    links.inject([]) do |collection, link_node|
      text = link_node.text.strip
      href = link_node.attributes["href"].value
      collection << Link.new(href, text, type)
      collection
    end
  end

end

class PathFix

  def initialize(root_path)
    @root_path = root_path
  end

  def make_paths_relative
    paths = Dir["#{@root_path}/**/*.html"]
    root_layers = @root_path.split("/").size
    paths.each do |path|
      layers = path.split("/").size - root_layers - 1
      content = File.read(path)

      content.gsub!(/href="\/\//, "href=\"http://")
      content.gsub!(/src="\/\//, "src=\"http://")
      content.gsub!(/value="\/\//, "value=\"http://")

      content.gsub!(/href="\//, "href=\"#{"../" * layers}")
      content.gsub!(/src="\//, "src=\"#{"../" * layers}")
      content.gsub!(/value="\//, "value=\"#{"../" * layers}")

      content.gsub!(/href=".*?\/#/, "href=\"#")

      File.open(path, "w+") do |f|
        f.write content
      end
    end
  end

  def make_paths_relative_for_root_path(prefix="")
    content = File.read(@root_path)

    content.gsub!('href="/', "href=\"#{prefix}")
    content.gsub!('src="/', "src=\"#{prefix}")
    content.gsub!('value="/', "value=\"#{prefix}")
    content.gsub!("url('/", "url('#{prefix}")
    content.gsub!(/url\(\/(.*?)\)/, "url('#{prefix}" + '\1' + "')")
    content.gsub!('url("/', "url(\"#{prefix}")

    File.open(@root_path, "w+") do |f|
      f.write content
    end
  end

  def complete_folder_references
    paths = Dir["#{@root_path}/**/*.html"]
    paths.each do |path|
      content = File.read(path)

      content.gsub!(/\/"/, "\/index.html\"")

      File.open(path, "w+") do |f|
        f.write content
      end
    end
  end

  def remove_codepen_references
    paths = Dir["#{@root_path}/**/*.html"]
    paths.each do |path|
      page = Nokogiri::HTML(open(path))
      page.css(".phone-case").remove

      File.open(path, "w+") do |f|
        f.write page.to_html
      end
    end
  end

  def cdn_to_statics(mapping)
    paths = Dir["#{@root_path}/**/*.html"]
    root_layers = @root_path.split("/").size
    mapping.each_pair do |cdn_path, static_path|
      paths.each do |path|
        layers = path.split("/").size - root_layers - 1
        content = File.read(path)

        content.gsub!(cdn_path, "#{"../" * layers}#{static_path}")

        File.open(path, "w+") do |f|
          f.write content
        end
      end
    end
  end

end

namespace :docset do

  task :generate => [ :cleanup,
                      :update_ionic_site,
                      :compile_ionic_site,
                      :create,
                      :complete_folder_references,
                      :remove_codepen_references,
                      :generate_docset_database,
                      :fix_href_in_main_index,
                      :copy_ionic_docs,
                      :make_paths_relative,
                      :cdn_to_static,
                      :fix_site_js,
                      :copy_static_files
                    ]

  desc "cleanup"
  task :cleanup do
    FileUtils.rm_rf("./compiled")
    FileUtils.rm_rf("./dist")
  end

  desc "checkout ionic-site"
  task :update_ionic_site do
    remote_repo = "https://github.com/driftyco/ionic-site.git"
    if File.exists?("./src")
      FileUtils.cd('./src') do
        `git pull`
      end
    else
      `git clone --depth=1 #{remote_repo} ./src`
    end
  end

  desc "compile with jekyll"
  task :compile_ionic_site do
    `jekyll build --layouts ./src/_layouts  --source ./src --destination ./compiled`
  end

  desc "creates the docset"
  task :create do
    FileUtils.mkdir_p("#{resources_path}/Documents")
  end

  desc "copies the bare ionic html documentation"
  task :copy_ionic_docs do
    dist = "#{resources_path}/Documents"
    FileUtils.mkdir_p("#{dist}/img")
    {
      "index.html" => "",
      "css" => "",
      "data" => "",
      "js" => "",
      "fonts" => "",
      "img/docs" => "img",
      "img/getting-started" => "img",
      "img/input-types" => "img",
      "img/*.png" => "img",
      "img/*.jpg" => "img",
      "img/*.svg" => "img",
      "img/testimonials" => "img",
      "img/homepage" => "img",
      "getting-started" => "",
      "docs" => "",
    }.each_pair do |src,dest|
      FileUtils.cp_r(Dir.glob("compiled/#{src}"), "#{dist}/#{dest}")
    end
    Dir["compiled/img/*.png", "compiled/img/*.svg"].each do |path|
      FileUtils.cp_r(path, "#{dist}/img/.")
    end
  end

  desc "copies Info.plist into the docset"
  task :copy_static_files do
    FileUtils.cp("Info.plist", "#{docset_root}/Contents/")
    FileUtils.cp("icon.png", "#{docset_root}")
  end

  desc "fixes the css and image paths"
  task :make_paths_relative do
    PathFix.new("#{resources_path}/Documents").make_paths_relative
  end

  desc "completes href references to folders with index.html"
  task :complete_folder_references do
    PathFix.new("compiled").complete_folder_references
  end

  desc "completes href references to folders with index.html"
  task :remove_codepen_references do
    PathFix.new("compiled").remove_codepen_references
  end

  desc "fix href references in main index.html"
  task :fix_href_in_main_index do
    PathFix.new("compiled/index.html").make_paths_relative_for_root_path
    PathFix.new("compiled/css/site.css").make_paths_relative_for_root_path("../")
  end

  desc "replaces cdn files with local files"
  task :cdn_to_static do
    FileUtils.cp_r("cdn", "#{resources_path}/Documents")
    PathFix.new("#{resources_path}/Documents").cdn_to_statics({
      'http://ajax.googleapis.com/ajax/libs/jquery/1.10.2/jquery.min.js' => 'cdn/jquery.min.js',
      'http://netdna.bootstrapcdn.com/bootstrap/3.0.2/js/bootstrap.min.js' => 'cdn/bootstrap.min.js',
      'http://cdnjs.cloudflare.com/ajax/libs/Cookies.js/0.4.0/cookies.min.js' => 'cdn/cookies.min.js',
      '"/js/site.js' => '"js/site.js',
    })
  end

  desc "fixes the site.js script"
  task :fix_site_js do
    site_js = File.read("#{resources_path}/Documents/js/site.js")

    pathname = "window.location.pathname.replace(/^.*?Contents\\/Resources\\/Documents/, '')"

    site_js.gsub!("[href=\"' + window.location.pathname", "[href*=\"' + #{pathname}")

    File.open("#{resources_path}/Documents/js/site.js", "w+") do |f|
      f.write site_js
    end
  end

  desc "create the sqlite3 index"
  task :generate_docset_database do
    sections_selector = ".menu-section a.api-section"
    link_selector     = ".menu-section ul a"
    js_generator = Generator.new("#{resources_path}/docSet.dsidx", "compiled/docs/nightly/index.html", sections_selector, link_selector)
    js_generator.parse
    js_generator.create_css_entry
    js_generator.build_css_tree
  end

end

task :default => ["docset:generate"]
